---
description: Start user-provided Erlang applications.
---

# Embedded Erlang Applications

NSO is capable of starting user-provided Erlang applications embedded in the same Erlang VM as NSO.

The Erlang code is packaged into applications which are automatically started and stopped by NSO if they are located at the proper place. NSO will search all packages for top-level directories called `erlang-lib`. The structure of such a directory is the same as a standard `lib` directory in Erlang. The directory may contain multiple Erlang applications. Each one must have a valid `.app` file. See the Erlang documentation of `application` and `app` for more info.

An Erlang package skeleton can be created by making use of the `ncs-make-package` command:

```
ncs-make-package --erlang-skeleton --erlang-application-name <appname> <package-name>
```

Multiple applications can be generated by using the option `--erlang-application-name NAME` multiple times with different names.

All application code should use the prefix `ec_` for module names, application names, registered processes (if any), and named `ets` tables (if any), to avoid conflict with existing or future names used by NSO itself.

## Erlang API <a href="#d5e1761" id="d5e1761"></a>

The Erlang API to NSO is implemented as an Erlang/OTP application called `econfd`. This application comes in two flavors. One is built into NSO to support applications running in the same Erlang VM as NSO. The other is a separate library which is included in source form in the NSO release, in the `$NCS_DIR/erlang` directory. Building `econfd` as described in the `$NCS_DIR/erlang/econfd/README` file will compile the Erlang code and generate the documentation.

This API can be used by applications written in Erlang in much the same way as the C and Java APIs are used, i.e. code running in an Erlang VM can use the `econfd` API functions to make socket connections to NSO for the data provider, MAAPI, CDB, etc. access. However, the API is also available internally in NSO, which makes it possible to run Erlang application code inside the NSO daemon, without the overhead imposed by the socket communication.

When the application is started, one of its processes should make initial connections to the NSO subsystems, register callbacks, etc. This is typically done in the `init/1` function of a `gen_server` or similar. While the internal connections are made using the exact same API functions (e.g. `econfd_maapi:connect/2`) as for an application running in an external Erlang VM, any `Address` and `Port` arguments are ignored, and instead, standard Erlang inter-process communication is used.

There is little or no support for testing and debugging Erlang code executing internally in NSO since NSO provides a very limited runtime environment for Erlang to minimize disk and memory footprints. Thus the recommended method is to develop Erlang code targeted for this by using `econfd` in a separate Erlang VM, where an interactive Erlang shell and all the other development support included in the standard Erlang/OTP releases are available. When development and testing are completed, the code can be deployed to run internally in NSO without changes.

For information about the Erlang programming language and development tools, refer to [www.erlang.org](https://www.erlang.org/) and the available books about Erlang (some are referenced on the website).

The `--printlog` option to `ncs`, which prints the contents of the NSO error log, is normally only useful for Cisco support and developers, but it may also be relevant for debugging problems with application code running inside NSO. The error log collects the events sent to the OTP error\_logger, e.g. crash reports as well as info generated by calls to functions in the error\_logger(3) module. Another possibility for primitive debugging is to run `ncs` with the `--foreground` option, where calls to `io:format/2` etc will print to standard output. Printouts may also be directed to the developer log by using `econfd:log/3`.

While Erlang application code running in an external Erlang VM can use basically any version of Erlang/OTP, this is not the case for code running inside NSO, since the Erlang VM is evolving and provides limited backward/forward compatibility. To avoid incompatibility issues when loading the `beam` files, the Erlang compiler `erlc` should be of the same version as was used to build the NSO distribution.

NSO provides the VM, `erlc` and the `kernel`, `stdlib`, and `crypto` OTP applications.

{% hint style="info" %}
Application code running internally in the NSO daemon can have an impact on the execution of the standard NSO code. Thus, it is critically important that the application code is thoroughly tested and verified before being deployed for production in a system using NSO.
{% endhint %}

## Application Configuration <a href="#d5e1797" id="d5e1797"></a>

Applications may have dependencies to other applications. These dependencies affect the start order. If the dependent application resides in another package, this should be expressed by using the required package in the `package-meta-data.xml` file. Application dependencies within the same package should be expressed in the `.app`. See below.

The following config settings in the `.app` file are explicitly treated by NSO:

<table data-header-hidden><thead><tr><th width="269"></th><th></th></tr></thead><tbody><tr><td><code>applications</code></td><td>A list of applications that need to be started before this application can be started. This info is used to compute a valid start order.</td></tr><tr><td><code>included_applications</code></td><td>A list of applications that are started on behalf of this application. This info is used to compute a valid start order.</td></tr><tr><td><code>env</code></td><td>A property list, containing <code>[{Key,Val}]</code> tuples. Besides other keys, used by the application itself, a few predefined keys are used by NSO. The key <code>ncs_start_phase</code> is used by NSO to determine which start phase the application is to be started in.  Valid values are <code>early_phase0</code>, <code>phase0</code>, <code>phase1</code>, <code>phase1_delayed</code> and <code>phase2</code>. Default is <code>phase1</code>. If the application is not required in the early phases of startup, set <code>ncs_start_phase</code> to <code>phase2</code> to avoid issues with NSO services being unavailable to the application. The key <code>ncs_restart_type</code> is used by NSO to determine which impact a restart of the application will have. This is the same as the <code>restart_type()</code> type in <code>application</code>. Valid values are <code>permanent</code>, <code>transient</code> and <code>temporary</code>. Default is <code>temporary</code>.</td></tr></tbody></table>

## Example <a href="#d5e1835" id="d5e1835"></a>

The `examples.ncs/getting-started/developing-with-ncs/18-simple-service-erlang` example in the bundled collection shows how to create a service written in Erlang and execute it internally in NSO. This Erlang example is a subset of the Java example `examples.ncs/getting-started/developing-with-ncs/4-rfs-service`.