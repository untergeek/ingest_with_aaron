# Ingest with Aaron: September 25, 2024

In today's session, we'll be going over how to ingest JSON data in both Logstash
and filebeat.

In [the filebeat README.md](filebeat/README.md), you'll find the installation
and configuration instructions for filebeat.

Likewise, in [the logstash README.md](logstash/README.md), you'll find the
installation and configuration instructions for Logstash.

The [samples](samples/) directory contains sample data referenced in the
configuration files, as well as in the live session.

The [logfiles](logfiles/) directory contains example data like that produced
during the live session.

## ANSI in the logfiles

The `rubydebug` codec used in the `file` output plugin in Logstash expands the
JSON structure and colors it using ANSI terminal codes in order to make it more
readable. This is great in the terminal, but much harder to read in, say, Visual
Studio Code.

In the live session, I will be using the [ANSI Colors](https://marketplace.visualstudio.com/items?itemName=iliazeus.vscode-ansi)
plugin for Visual Studio Code to make it easier to view these.
