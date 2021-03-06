# Phoeβe

![CI](https://github.com/SUSE/phoebe/workflows/CI/badge.svg)


## Idea

Phoeβe (/ˈfiːbi/) wants to add basic artificial intelligence capabilities to the Linux OS.


## What problem Phoeβe wants to solve

System-level tuning is a very complex activity, requiring the knowledge and expertise of several (all?) layers which compose
the system itself, how they interact with each other and (quite often) it is required to also have an intimate knowledge of
the implementation of the various layers.

Another big aspect of running systems is dealing with failure. Do not think of failure as a machine turning on fire rather as 
an overloaded system, caused by misconfiguration, which could lead to starvation of the available resources.

In many circumstances, operators are used to deal with telemetry, live charts, alerts, etc. which could help them identifying 
the offending machine(s) and (re)act to fix any potential issues.

However, one question comes to mind: wouldn't it be awesome if the machine could auto-tune itself and provide a self-healing 
capability to the user? Well, if that is enough to trigger your interest then this is what Phoeβe aims to provide.

Phoeβe uses system telemetry as the input to its <i>brain</i> and produces a big set of settings which get applied to the 
running system. The decision made by the <i>brain</i> is continuously reevaluated (considering the <i>grace_period</i> setting) 
to offer eventually the best possible setup.


## Architecture

Phoeβe is designed with a plugin architecture in mind, providing an interface for new functionality to be added with ease.

Plugins are loaded at runtime and registered with the main body of execution. The only requirement is to implement the interface dictated by the structure *plugin_t*. The **network_plugin.c** represents a very good example of how to implement a new plugin for Phoeβe.


## Disclaimer

The mathematical model implemented is a super-basic one, which implements a *machine-learning 101* approach:
**input * weight + bias**. It does not use any fancy techniques and the complexity is close to zero.

The plan is to eventually migrate towards a model created in Tensorflow and exported so to be used by Phoeβe, but
we are not there yet.


## 10,000 feet view

The code allows for both **training** and **inference:** — all the knobs which can
modify the run-time behavior of the implementation are configurable via the *settings.json* file,
where each parameter is explained in detail.


For the inference case, when a match is found, then the identified kernel parameters are configured accordingly.

The inference loop runs every N seconds and the value is configurable via the **inference_loop_period**.
Depending on how quick we want the system to react to a situation change, then the value given to the
**inference_loop_period** will be bigger or smaller.

The code has a dedicated stats collection thread which periodically collects system statistics and populates structures
used by the inference loop. The statistics are collected every _N_ seconds, and this value is configurable via the
**stats_collection_period**. Depending on the overall network demands, the value of
**stats_collection_period** will be bigger or smaller to react slower or quicker to network events.


In case a high traffic rate is seen on the network and a matching entry is found, then the code will not consider
any lower values for a certain period of time: the value is configurable via the **grace_period** in
the *settings.json* file.

That behavior has been implemented to avoid causing too much reconfiguration on the system and to prevent
sudden system reconfiguration due to network spikes.

The code also supports few approximation functions, also available via the *settings.json* file.

The approximation functions can tune the tolerance value - runtime calculated - to further allow the user for fine
tuning of the matching criteria. Depending on the approximation function, obviously, the matching criteria could be
narrower or broader.


## Settings

Below is a detailed an explanation of what configurations are available in settings.json, the possible values and what effect they have. (Note that this is not really valid JSON; please remove the lines with double-forward-slashes if you use it.)

```jsonc
{
    "app_settings": {

        // path where application is expecting to find plugins to load
        "plugins_path": "/home/mvarlese/REPOS/ai-poc/bin",

        // max_learning_values: number of values learnt per iteration
        "max_learning_values": 1000,

        // save trained data to file every saving_loop value
        "saving_loop": 10,

        // accuracy: the level of accuracy to find a potential entry
        // given the transfer rate considered.
        //
        // MaxValue: Undefined, MinValue: 0.00..1
        // Probably not very intuitive: a higher number correspondes to
        // a higher accuracy level.
        "accuracy": 0.5,

        // approx_function: the approximation function applied
        // to the calculated tolerance value used to find a
        // matching entry in values.
        //
        // Possible values:
        // 0 = no approximation function
        // 1 = square-root
        // 2 = power-of-two
        // 3 = log10
        // 4 = log
        "approx_function": 0,

        // grace_period: the time which must be elapsed
        // before applying new settings for a lower
        // transfer rate than the one previously measured.
        "grace_period": 10,

        // stats_loop_period: the cadence of time which
        // has to be elapsed between stats collection.
        // It is expressed in seconds but it accepts non-integer
        // values; ie. 0.5 represents half-second
        "stats_collection_period": 0.5,

        // inferece_loop_period: the time which must be
        // elapsed before running a new inference evaluation
        "inference_loop_period": 1

    },

    "weights":{
        "transfer_rate_weight": 0.8,
        "drop_rate_weight" : 0.1,
        "errors_rate_weight" : 0.05,
        "fifo_errors_rate_weight" : 0.05
    },

    "bias": 10
}
```

## Building

The PoC code is built using [Meson](https://mesonbuild.com/):

```ShellSession
$ meson build
$ cd build/
$ meson compile
```

You can also run debug builds using address or undefined behavior sanitizer:

```ShellSession
$ meson build -Db_sanitize=undefined # or -Db_sanitize=address for ASAN
$ cd build/
$ meson compile
```

There are few compile-time flags which can be passed to Meson to enable some code behavior:

* **print_messages**: used to print to `stdout` only the most important messages (this is the only parameter enabled by default)
* **print_advanced_messages**: used for very verbose printing to `stdout` (useful for debugging purposes)
* **print_table**: used to print to `stdout` all data stored in the different tables maintained by the application
* **apply_changes**: this enables the application to actually apply the settings via `sysctl`/`ethtool` command
* **check_initial_settings**: when enabled, this will prevent the application from applying lower settings than the ones already applied to the system at bootstrap
* **m_threads**: when enabled, this will run training using as many threads as available cores on the machine

These flags can be enabled by passing them to the Meson configure step:

```ShellSession
$ meson -Dprint_advanced_messages=true -Dprint_table=true build
```


## Running
The code supports multiple mode of operation:

* Training mode:
```ShellSession
./build/src/phoebe -f ./csv_files/rates_trained_data.csv -m training -s settings.json
```

* Inference
```ShellSession
./build/src/phoebe -f ./csv_files/rates_trained_data.csv -i wlan0 -m inference -s settings.json
```


## Feedback / Input / Collaboration
<p>
If you are curious about the project and want more information, please, do reach out to <a href="mailto:marco.varlese@suse.com">marco.varlese@suse.com</a>.<br>
I will be more than happy to talk to you more about this project and what other initiatives are in this area.
</p>
