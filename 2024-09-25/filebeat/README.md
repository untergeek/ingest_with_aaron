# Install, configure, and run Filebeat

## Download

Visit https://www.elastic.co/downloads/beats/filebeat to get the proper download
for _your_ OS and hardware architecture. For me, that's an M-series MacBook, so
I download the `darwin-aarch64` version. 

### Recommendation

For this exercise, I recommend using the tarball on a MacOS or Linux device, or
the Zip install for Windows, as the instructions I'm using call for direct execution
of `filebeat`. This doesn't mean it's impossible to run this exercise from an MSI
Windows install, or from an RPM or DEB package install on Linux. It's just messier
since those are designed to start and run automatically as a service.

Non-package versions include:

* [Linux x86_64](https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.15.1-linux-x86_64.tar.gz)
* [Linux aarch64](https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.15.1-linux-arm64.tar.gz)
* [macOS x86_64](https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.15.1-darwin-x86_64.tar.gz)
* [macOS aarch64](https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.15.1-darwin-aarch64.tar.gz)
* [Windows x86_64](https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-8.15.1-windows-x86_64.zip)

I recommend making a dedicated directory somewhere that this can go inside. I use
`$HOME/Ingest` as I work with multiple different versions of Elastic's products.
Whatever works for you should be fine, but it shouldn't be nested too deeply.
(You're on your own to manually download for Windows).

```
mkdir -p $HOME/Ingest/samples
cd $HOME/Ingest
# Pick one: 
#   wget $LINK
#   curl -Os $LINK
```

Replace `$LINK` with your chosen download URL.


## Install

Uncompress the tarball:

```
tar zxf filebeat-8.15.1-$OS-$ARCH.tar.gz
```

Replace `$OS` and `$ARCH` with whatever your file is.

## Configure

Navigate to the new `filebeat-8.15.1-$OS-$ARCH` directory.

Backup the bundled `filebeat.yml` to `filebeat.yml-orig`:

```
mv filebeat.yml filebeat-yml.orig
touch filebeat.yml
```

You may wish to `cp` the file instead. I move/rename it, as it's easier in
this exercise to start with a blank file and add only what we will be using.

### `filebeat.yml`

Now, you can edit these however you like. `vi`, `emacs`, Visual Studio Code, any
way you like, provided it doesn't do Rich Text type editing and mess up your format.

```yaml
---
filebeat.inputs:
- type: filestream
  id: ingest-with-aaron
  enabled: true
  paths:
    - $HOME/Ingest/samples/filebeat_by_line

output.logstash:
  hosts: ["localhost:5044"]

processors:
  - decode_json_fields:
      fields: ["message"]
      process_array: false
      max_depth: "1"
      target: ""
      overwrite_keys: false
      add_error_key: true
  - decode_json_fields:
      fields: ["json"]
      process_array: false
      max_depth: "2"
      target: ""
      overwrite_keys: false
      add_error_key: true
  - drop_fields:
      when:
        has_fields: ["json", "message"]
      fields: ["json", "message"]
      ignore_missing: false

# These are not for performance, but for lower latency during the presentation
# DO NOT USE THESE SETTINGS IN YOUR ENVIRONMENT
queue.mem.flush.timeout: 1s
idle_connection_timeout: 60s
worker: 1
bulk_max_size: 10
```

## Run filebeat

Running filebeat is trickier than it seems as it likes to know what paths it
should use. When installing and executing from a packaged install, e.g. RPM, DEB,
or Windows MSI, these settings are configured and managed automatically so all
you would need to do is `systemctl start filebeat.service`, for example. Since we
are running from a local path, we provide all of these settings manually:

```
./filebeat -c $(pwd)/filebeat.yml -path.config $(pwd) -path.data $(pwd)/data -path.logs $(pwd)/logs
```

That might be too long to easily view in GitHub, so here it is with escaped line
breaks:

``` 
./filebeat \
  -c $(pwd)/filebeat.yml \
  -path.config $(pwd) \
  -path.data $(pwd)/data \
  -path.logs $(pwd)/logs
```

### The filebeat registry

If you are repeatedly re-ingesting a particular log file, you'll find that the
filebeat registry will have stored that information and will refuse to re-read the
log because "it was already seen."  In this case, I pre-pend the above statement
with an `rm -rf $(pwd)/data/* ; ` to clear out the registry each time:

```
rm -rf $(pwd)/data/* ; ./filebeat -c $(pwd)/filebeat.yml -path.config $(pwd) -path.data $(pwd)/data -path.logs $(pwd)/logs
```

Since that may be harder to read as a single line on GitHub, here it is with line
continuing escapes:

```
rm -rf $(pwd)/data/* ; \
./filebeat \
  -c $(pwd)/filebeat.yml \
  -path.config $(pwd) \
  -path.data $(pwd)/data \
  -path.logs $(pwd)/logs
```

I expressly do not use this approach in this exercise because I am continuously
appending new lines to the file we are reading. If I deleted the registry each
time, I would be re-reading the entire file each time. That would potentially be
a lot of extra lines being read for no good reason.

## Add sample data

Now that we have filebeat running in a terminal window, we can add our test logs
by appending to the file our `filestream` input is configured to read:

```
  paths:
    - $HOME/Ingest/samples/filebeat_by_line
```

So this is as simple as either using `echo` or `cat`, and using the `>>` append
redirector:

### echo

```
echo '{JSON_STRING}' >> $HOME/Ingest/samples/filebeat_by_line
```

Of course, `{JSON_STRING}` should be the full, one-line value of the JSON you want
to append.

### cat

```
cat $HOME/Ingest/samples/FILENAME.json >> $HOME/Ingest/samples/filebeat_by_line
```

Replace `FILENAME` with the name of the file you're appending.

Take care that if you're appending an entire file that you're not sending too many
lines at once. In such a case, you may want to use `tail`, `head`, or `grep` to
only send one, or a few lines at a time:

```
tail -1 $HOME/Ingest/samples/FILENAME.json >> $HOME/Ingest/samples/filebeat_by_line
head -1 $HOME/Ingest/samples/FILENAME.json >> $HOME/Ingest/samples/filebeat_by_line
grep 'SEARCH STRING' $HOME/Ingest/samples/FILENAME.json >> $HOME/Ingest/samples/filebeat_by_line
```
