FROM google/dart:2.9

WORKDIR /opt/web

{{#files}}
    COPY {{{.}}} {{{.}}}
{{/files}}

RUN pub get

CMD dart main.dart
