# Install, configure, and run Logstash

## Download

Visit https://www.elastic.co/downloads/logstash to get the proper download
for _your_ OS and hardware architecture. For me, that's an M-series MacBook, so
I download the `darwin-aarch64` version. 

### Recommendation

For this exercise, I recommend using the tarball on a MacOS or Linux device, or
the Zip install for Windows, as the instructions I'm using call for direct execution
of `bin/logstash`. This doesn't mean it's impossible to run this exercise from an
RPM or DEB package install on Linux. It's just messier since those are designed
to start and run automatically as a service.

Non-package versions include:

* [Linux x86_64](https://artifacts.elastic.co/downloads/logstash/logstash-8.15.1-linux-x86_64.tar.gz)
* [Linux aarch64](https://artifacts.elastic.co/downloads/logstash/logstash-8.15.1-linux-aarch64.tar.gz)
* [macOS x86_64](https://artifacts.elastic.co/downloads/logstash/logstash-8.15.1-darwin-x86_64.tar.gz)
* [macOS aarch64](https://artifacts.elastic.co/downloads/logstash/logstash-8.15.1-darwin-aarch64.tar.gz)
* [Windows x86_64](https://artifacts.elastic.co/downloads/logstash/logstash-8.15.1-windows-x86_64.zip)

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
tar zxf logstash-8.15.1-$OS-$ARCH.tar.gz
```

Replace `$OS` and `$ARCH` with whatever your file is.

## Configure

Navigate to the new `logstash-8.15.1-$OS-$ARCH/config` directory.

You may wish to `cp` the file instead. I move/rename it, as it's easier in
this exercise to start with a blank file and add only what we will be using.

### logstash.yml

Confirm you're in the `config` path, and rename the bundled `logstash.yml` to
`logstash.yml-orig`:

```
mv logstash.yml logstash-yml.orig
touch logstash.yml
```

Edit this file and put in these lines:

```yaml
---
config.reload.automatic: "true"
config.reload.interval: "3s"
pipeline.ordered: "false"
```

Yep! That's it for this kind of testing!

### pipelines.yml

Confirm you're in the `config` path, and rename the bundled `pipelines.yml` to
`pipelines.yml-orig`:

```
mv pipelines.yml pipelines-yml.orig
touch pipelines.yml
```

Edit `pipelines.yml` and put in these lines:

```yaml
---
# Reading JSON from a file
- pipeline.id: from_json
  pipeline.workers: 1
  pipeline.batch.size: 1
  path.config: "../pipelines/from_json.conf"
# Receiving data from filebeat
- pipeline.id: from_beats
  pipeline.workers: 1
  pipeline.batch.size: 1
  path.config: "../pipelines/from_beats.conf"
```

### Create the `$HOME/Ingest/pipelines` directory

```
mkdir -p $HOME/Ingest/pipelines
cd $HOME/Ingest/pipelines
```

Now that we're here, we can create our configuration files:

```
touch $HOME/Ingest/pipelines/from_beats.conf
touch $HOME/Ingest/pipelines/from_json.conf
```

Note that these file names match those in the `path.config` lines of our
`pipelines.yml` file, though those are _relative_ paths. This is acceptable!
`$HOME/Ingest` is up one level from `$HOME/Ingest/logstash`, and then `pipelines`
is a subdirectory of `Ingest`, making `../pipelines/FILENAME.conf` a good way
to preserve your pipelines _outside_ of your versioned `logstash` installation.

### from_json.conf

Fire up your editor and add these lines to `from_json.conf`:

```
input {
  file {
    # In Logstash's file input, you MUST replace $HOME with the actual path.
    # Logstash will yield an error if you use a non-absolute path:
    # <ArgumentError: File paths must be absolute, relative path specified: $HOME/Ingest/samples/read_by_line>
    path => [ "$HOME/Ingest/samples/read_by_line" ]
    codec => json { target => "[@metadata]" }
  }
}

filter {
  json { source => "[@metadata][json]" }
  mutate { remove_field => [ "event", "host", "log", "@version", "@timestamp" ] }
}

output {
  file {
    path => "../logfiles/from_json.ansi"
    codec => rubydebug { metadata => true }
  }
}
```

## Run Logstash

You can run Logstash from its own directory, or from `$HOME/Ingest`. We want it
to log to STDOUT so we can watch what's happening, so we will run it like this:

```
cd $HOME/Ingest
logstash-8.15.1-$OS-$ARCH/bin/logstash
```

or like this:

```
cd $HOME/Ingest/logstash-8.15.1-$OS-$ARCH
bin/logstash
```

Either is acceptable. You should see quite a bit of logging data generated in
your terminal window, but it should eventually say something like:

```
[2024-09-25T10:26:25,885][INFO ][logstash.agent           ] Pipelines running {:count=>2, :running_pipelines=>[:from_beats, :from_json], :non_running_pipelines=>[]}
```

### Logstash's file input sincedb

If you are repeatedly re-ingesting a particular log file, you'll find that the
`sincedb` will have stored that information and will refuse to re-read the log
because "it was already seen."  In cases like this, I preemptively will prevent
the `sincedb` from storing any data by setting its location to `/dev/null`:

```
  file {
    path => [ "$HOME/Ingest/samples/read_by_line" ]
    codec => json { target => "[@metadata]" }
    sincedb_path => "/dev/null"
  }
```

I expressly do not use this approach in this exercise because I am continuously
appending new lines to the file we are reading. If I deleted the registry each
time, I would be re-reading the entire file each time. That would potentially be
a lot of extra lines being read for no good reason.

## Add sample data

Now that we have Logstash running in a terminal window, we can add our test logs
by appending to the file our `file` input is configured to read:

**REMEMBER:** `$HOME` is only here as a placeholder. Logstash needs absolute
paths here.

```
path => [ "$HOME/Ingest/samples/read_by_line" ]
```

So this is as simple as either using `echo` or `cat`, and using the `>>` append
redirector:

### echo

```
echo '{JSON_STRING}' >> $HOME/Ingest/samples/read_by_line
```

Of course, `{JSON_STRING}` should be the full, one-line value of the JSON you want
to append.

### cat

```
cat $HOME/Ingest/samples/FILENAME.json >> $HOME/Ingest/samples/read_by_line
```

Replace `FILENAME` with the name of the file you're appending.

Take care that if you're appending an entire file that you're not sending too many
lines at once. In such a case, you may want to use `tail`, `head`, or `grep` to
only send one, or a few lines at a time:

```
tail -1 $HOME/Ingest/samples/FILENAME.json >> $HOME/Ingest/samples/read_by_line
head -1 $HOME/Ingest/samples/FILENAME.json >> $HOME/Ingest/samples/read_by_line
grep 'SEARCH STRING' $HOME/Ingest/samples/FILENAME.json >> $HOME/Ingest/samples/read_by_line
```
