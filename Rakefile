# frozen_string_literal: true

require 'yaml'
require 'mustache'
require 'dotenv'
require 'droplet_kit'
require 'net/ssh'
require 'net/scp'
require 'net/http'
require 'fileutils'
require 'base64'

environment = ENV.fetch('ENV', 'development')

MANIFESTS = {
  container: '.Dockerfile',
  build: '.Makefile'
}.freeze

default_environment = File.join('.env', 'default')
custom_environment = File.join('.env', environment)
Dotenv.load(custom_environment, default_environment)

class ::Hash
  def recursive_merge(h)
    merge!(h) { |_key, _old, _new| _old.instance_of?(Hash) ? _old.recursive_merge(_new) : _new }
  end
end

def default_provider
  if RbConfig::CONFIG['host_os'] =~ /linux/
    'docker'
  else
    'docker-machine'
  end
end

def commands_for(language, framework, **options)
  config = YAML.safe_load(File.read('config.yaml'))

  directory = File.dirname(options[:path])
  main_config = YAML.safe_load(File.open(File.join(directory, '..', '..', 'config.yaml')))
  language_config = YAML.safe_load(File.open(File.join(directory, '..', 'config.yaml')))
  framework_config = YAML.safe_load(File.open(File.join(directory, 'config.yaml')))
  app_config = main_config.recursive_merge(language_config).recursive_merge(framework_config)

  options[:framework] = framework
  options[:language] = language

  ENV.each do |key, value|
    options[key] = value unless options.key?(key)
  end

  commands = []

  # Compile first, only for non containers

  if app_config.key?('binaries') && !(options[:provider].start_with?('docker') || options[:provider].start_with?('podman'))
    commands << "docker build -f #{MANIFESTS[:container]} -t #{language}.#{framework} ."
    commands << "docker run -td #{language}.#{framework} > cid.txt"
    app_config['binaries'].each do |out|
      if out.count(File::Separator).positive?
        FileUtils.mkdir_p(File.join(directory, File.dirname(out)))
        commands << "docker cp `cat cid.txt`:/opt/web/#{File.dirname(out)} ."
      else
        commands << "docker cp `cat cid.txt`:/opt/web/#{out} #{out}"
      end
    end
  end

  config['providers'][options[:provider]]['build'].each do |cmd|
    commands << Mustache.render(cmd, options.merge!(manifest: MANIFESTS[:container])).to_s
  end

  config['providers'][options[:provider]]['metadata'].each do |cmd|
    commands << Mustache.render(cmd, options).to_s
  end

  if app_config.key?('bootstrap') && config['providers'][options[:provider]].key?('exec')
    remote_command = config['providers'][options[:provider]]['exec']
    app_config['bootstrap'].each do |cmd|
      commands << Mustache.render(remote_command, options.merge!(command: cmd)).to_s
    end
  end

  if config['providers'][options[:provider]].key?('reboot')
    commands << config['providers'][options[:provider]].fetch('reboot')
    commands << 'sleep 30'
  end

  commands << 'curl --retry 5 --retry-delay 5 --retry-max-time 180 --retry-connrefused http://`cat ip.txt`:3000 -v'

  commands << "DATABASE_URL=#{ENV['DATABASE_URL']} ../../bin/client --language #{language} --framework #{framework} #{options[:sieger_options]} -h `cat ip.txt`" unless options[:collect] == 'off'

  unless options[:clean] == 'off'
    config['providers'][options[:provider]]['clean'].each do |cmd|
      commands << Mustache.render(cmd, options).to_s
    end
  end

  commands
end

def create_dockerfile(language, framework, **options)
  directory = File.join(Dir.pwd, language, framework)
  main_config = YAML.safe_load(File.open(File.join(Dir.pwd, 'config.yaml')))
  language_config = YAML.safe_load(File.open(File.join(Dir.pwd, language, 'config.yaml')))
  framework_config = YAML.safe_load(File.open(File.join(directory, 'config.yaml')))
  config = main_config.recursive_merge(language_config).recursive_merge(framework_config)

  if config.key?('sources')
    files = []
    config['sources'].each do |path|
      Dir.glob(File.join(directory, path)).each do |f|
        if f =~ /^*\.\./
          filename = f.gsub(directory, '').gsub!(%r{/\.\./\.}, '')
          File.open(File.join(directory, filename), 'w') { |stream| stream.write(File.read(f)) }
          files << filename
        else
          files << f.gsub!(directory, '').gsub!(%r{^/}, '')
        end
      end
    end
    config['sources'] = files
  end
  if config.key?('files')
    files = []
    config['files'].each do |path|
      Dir.glob(File.join(directory, path)).each do |f|
        if f =~ /^*\.\./
          filename = f.gsub(directory, '').gsub!(%r{/\.\./\.}, '')
          File.open(File.join(directory, filename), 'w') { |stream| stream.write(File.read(f)) }
          files << filename
        else
          files << f.gsub!(directory, '').gsub!(%r{^/}, '')
        end
      end
    end
    config['files'] = files
  end

  template = nil
  if options[:provider].start_with?('docker') || options[:provider].start_with?('podman')
    template = File.join(directory, '..', 'Dockerfile')
  elsif config.key?('binaries')
    template = File.join(directory, '..', '.build', options[:provider], 'Dockerfile')
  end

  if config.key?('environment')
    environment = []
    config.fetch('environment').each do |key, value|
      environment << "#{key} #{value}"
    end
    config['environment'] = environment
  end

  File.open(File.join(directory, MANIFESTS[:container]), 'w') { |f| f.write(Mustache.render(File.read(template), config)) } if template
end

task :config do
  provider = ENV.fetch('PROVIDER') { default_provider }
  collect = ENV.fetch('COLLECT', 'on')

  sieger_options = ENV.fetch('SIEGER_OPTIONS', '-r GET:/ -c 10')
  clean = ENV.fetch('CLEAN', 'on')

  Dir.glob('*/*/config.yaml').each do |path|
    directory = File.dirname(path)
    language, framework = directory.split(File::Separator)

    create_dockerfile(language, framework, provider: provider)

    makefile = File.open(File.join(language, framework, MANIFESTS[:build]), 'w')

    makefile.write("build:\n")

    commands_for(language, framework, provider: provider, clean: clean, sieger_options: sieger_options, path: path, collect: collect).each do |command|
      makefile.write("\t #{command}\n")
    end

    makefile.close
  end
end

namespace :cloud do
  task :config do
    language = ENV.fetch('LANG')
    framework = ENV.fetch('FRAMEWORK')

    directory = File.join(Dir.pwd, language, framework)
    main_config = YAML.safe_load(File.open(File.join(Dir.pwd, 'config.yaml')))
    language_config = YAML.safe_load(File.open(File.join(Dir.pwd, language, 'config.yaml')))
    framework_config = YAML.safe_load(File.open(File.join(directory, 'config.yaml')))
    config = main_config.recursive_merge(language_config).recursive_merge(framework_config)

    config['cloud']['config']['write_files'] = if config.key?('service')
                                                 [{
                                                   'path' => '/usr/lib/systemd/system/web.service',
                                                   'permission' => '0644',
                                                   'content' => Mustache.render(config['service'], config)
                                                 }]
                                               else
                                                 []
                                               end

    if config.key?('environment')
      environment = config.fetch('environment')
      stringified_environment = String.new
      environment.map { |k, v| stringified_environment += "#{k}=#{v}\n" }
      config['cloud']['config']['write_files'] << {
        'path' => '/etc/web',
        'permission' => '0644',
        'content' => stringified_environment
      }
    else
      config['cloud']['config']['write_files'] << {
        'path' => '/etc/web',
        'permission' => '0644',
        'content' => ''
      }
    end

    if config.key?('deps')
      config['cloud']['config']['packages'] = [] unless config['cloud']['config'].key?('packages')
      config['deps'].each do |package|
        config['cloud']['config']['packages'] << package
      end
    end

    if config.key?('before_command')
      commands = config['cloud']['config']['runcmd'] || []
      config['cloud']['config']['runcmd'] = []
      config['before_command'].each do |cmd|
        config['cloud']['config']['runcmd'] << cmd
      end
      commands.each do |cmd|
        config['cloud']['config']['runcmd'] << cmd
      end
    end

    if config.key?('php_ext')
      config['php_ext'].each do |deps|
        config['cloud']['config']['runcmd'] << "pecl install #{deps}"
        config['cloud']['config']['runcmd'] << "echo 'extension=#{deps}' > /etc/php.d/99-#{deps}.ini"
      end
    end

    if config.key?('after_command')
      config['after_command'].each do |cmd|
        config['cloud']['config']['runcmd'] << cmd
      end
    end

    if config.key?('files')
      config['files'].each do |pattern|
        path = File.join(directory, pattern)
        files = Dir.glob(path)

        files.each do |path|
          remote_path = path.gsub(directory, '').gsub(%r{^/}, '').gsub(%r{^\.\./\.}, '')
          remote_directory = File.dirname(remote_path)

          config['cloud']['config']['write_files'] << {
            'path' => "/opt/web/#{remote_path}",
            'content' => File.read(path),
            'permission' => '0644'
          }

          next if remote_directory.start_with?('.')
        end
      end
    end

    File.open(File.join(directory, 'user_data.yml'), 'w') do |f|
      f.write('#cloud-config')
      f.write("\n")
      f.write(config['cloud']['config'].to_yaml)
    end
  end

  task :upload do
    language = ENV.fetch('LANG')
    framework = ENV.fetch('FRAMEWORK')

    directory = File.join(Dir.pwd, language, framework)
    main_config = YAML.safe_load(File.open(File.join(Dir.pwd, 'config.yaml')))
    language_config = YAML.safe_load(File.open(File.join(Dir.pwd, language, 'config.yaml')))
    framework_config = YAML.safe_load(File.open(File.join(directory, 'config.yaml')))
    config = main_config.recursive_merge(language_config).recursive_merge(framework_config)

    if config.key?('binaries')
      binaries = []
      config['binaries'].each do |pattern|
        Dir.glob(File.join(directory, pattern)).each do |binary|
          binaries << binary
        end
      end

      Net::SSH.start(ENV['HOST'], 'root', keys: [ENV['SSH_KEY']]) do |ssh|
        binaries.each do |binary|
          remote_directory = File.dirname(binary).gsub!(directory, '/opt/web')
          $stdout.puts "Creating #{remote_directory}"
          ssh.exec!("mkdir -p #{remote_directory}")
        end
      end

      Net::SCP.start(ENV['HOST'], 'root', keys: [ENV['SSH_KEY']]) do |scp|
        config['binaries'].each do |pattern|
          Dir.glob(File.join(directory, pattern)).each do |binary|
            remote_directory = File.dirname(binary).gsub!(directory, '/opt/web')
            $stdout.puts "Uploading #{binary} to #{remote_directory}"
            scp.upload!(binary, remote_directory, verbose: true, recursive: true)
          end
        end
      end
    end
  end

  task :wait do
    while true
      $stdout.puts 'Tring to connect'
      begin
        ssh = Net::SSH.start(ENV['HOST'], 'root', keys: [ENV['SSH_KEY']])
      rescue Errno::ECONNREFUSED, Errno::EHOSTUNREACH
        sleep 5
        next
      else
        break
      end
    end

    loop do
      output = ssh.exec!('cloud-init status')
      _, status = output.split(':')

      raise 'Cloud-init have failed' if status.strip == 'error'

      break if status.strip == 'done'

      $stdout.puts 'Cloud-init is still running'
      sleep 5
    end

    ssh.close
  end
end

namespace :ci do
  task :config do
    blocks = [{ name: 'setup', dependencies: [], task: {
      jobs: [{
        name: 'setup',
        commands: [
          'checkout',
          'cache store $SEMAPHORE_GIT_SHA .',
          'sudo snap install crystal --classic',
          'sudo apt-get -y install libyaml-dev libevent-dev',
          'shards build --static',
          'cache store bin bin',
          'bundle config path .cache',
          'bundle install',
          'cache store built-in .cache',
          'bundle exec rake config'
        ]
      }]
    } }]
    Dir.glob('*/config.yaml').each do |path|
      language, = path.split(File::Separator)
      block = { name: language, dependencies: ['setup'], task: { prologue: { commands: [
        'cache restore $SEMAPHORE_GIT_SHA',
        'cache restore bin',
        'cache restore built-in',
        'find bin -type f -exec chmod +x {} \\;',
        'bundle config path .cache',
        'bundle install',
        'bundle exec rake config'
      ] }, 'env_vars': [
        { name: 'CLEAN', value: 'off' },
        { name: 'COLLECT', 'value': 'off' }
      ], jobs: [] } }
      Dir.glob("#{language}/*/config.yaml") do |file|
        _, framework, = file.split(File::Separator)
        block[:task][:jobs] << { name: framework, commands: [
          "cd #{language}/#{framework} && make build  -f #{MANIFESTS[:build]}  && cd -",
          "FRAMEWORK=#{language}/#{framework} bundle exec rspec .spec"
        ] }
      end
      blocks << block
    end

    config = { version: 'v1.0', name: 'Benchmarking suite', execution_time_limit: { hours: 3 }, agent: { machine: { type: 'e1-standard-2', os_image: 'ubuntu1804' } }, blocks: blocks }
    File.write('.semaphore/semaphore.yml', JSON.parse(config.to_json).to_yaml)
  end
end

task :clean do
  Dir.glob('**/.gitignore').each do |ignore_file|
    directory = File.dirname(ignore_file)
    next if directory.start_with?('lib')
    next if directory.start_with?('bin')

    File.foreach(ignore_file) do |line|
      line.strip!
      next if line.start_with?('!')
      next if line.start_with?('#')
      next if line.start_with?('.env')
      next if line.empty?

      Dir.glob(File.join(directory, line)).each do |file|
        if File.exist?(file)
          if File.file?(file)
            warn "Delting file #{file}"
            File.delete(file)
          elsif File.directory?(file)
            warn "Deleting directory #{file}"
            FileUtils.rm_rf(file)
          end
        end
      end
    end
  end
end
