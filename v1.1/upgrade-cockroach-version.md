---
title: Upgrade a Cluster's Version
summary: Learn how to upgrade your CockroachDB cluster to a new version.
toc: false
toc_not_nested: true
---

Because of CockroachDB's multi-active availability design, you can perform a "rolling upgrade" of CockroachDB on your cluster. This means you can upgrade individual nodes in your cluster one at a time without any downtime for your cluster.

This page shows you how to upgrade a cluster from v1.0.x to v1.1, or from v1.1 to a patch release in the 1.1.x series. To upgrade within the 1.0.x series, see [the 1.0 version of this page](../v1.0/upgrade-cockroach-version.html).

<div id="toc"></div>

## Step 1. Prepare to upgrade

Before starting the upgrade, complete the following steps.

1. Make sure your cluster is behind a load balancer, or your clients are configured to talk to multiple nodes. If your application communicates with a single node, stopping that node to upgrade its CockroachDB binary will cause your application to fail.

2. Verify that all nodes are live by running the [`cockroach node status`](view-node-details.html) command with the `--decommission` flag against any node in the cluster.

    In the response, if `is_live` is `false` for any nodes that should be live, identify why the nodes are offline and restart them before begining your upgrade.

3. Verify the cluster's overall health and version by running the [`cockroach node status`](view-node-details.html) command with the `--ranges` flag against any node in the cluster.

    In the response, make sure `ranges_unavailable` and `ranges_underreplicated` show `0` for all nodes. If there are unavailable or underreplicated ranges in your cluster, performing a rolling upgrade increases the risk that ranges will lose a majority of their replicas and cause cluster unavailability. Therefore, it's important to identify and resolve the cause of range unavailability and underreplication before beginning your upgrade.

    Also, make sure the `build` field shows the same version of CockroachDB for all nodes. If any nodes are behind, upgrade them to the cluster's current version first, and then start this process over.

4. Capture the cluster's current state by running the [`cockroach debug zip`](debug-zip.html) command against any node in the cluster. If the upgrade does not go according to plan, the captured details will help you and Cockroach Labs troubleshoot issues.

5. [Back up the cluster](back-up-data.html). If the upgrade does not go according to plan, you can use the data to restore your cluster to its previous state.

## Step 2. Perform the rolling upgrade

For each node in your cluster, complete the following steps.

{{site.data.alerts.callout_success}}We recommend creating scripts to perform these steps instead of performing them by hand.{{site.data.alerts.end}}

{{site.data.alerts.callout_danger}}Upgrade only one node at a time, and wait at least one minute after a node rejoins the cluster to upgrade the next node. Simultaneously upgrading more than one node increases the risk that ranges will lose a majority of their replicas and cause cluster unavailability.{{site.data.alerts.end}}

1. Connect to the node.

2. Download and install the CockroachDB binary you want to use:
    - **Mac**:

        {% include copy-clipboard.html %}
        ~~~ shell
        # Get the CockroachDB tarball:
        $ curl -O https://binaries.cockroachdb.com/cockroach-{{page.release_info.version}}.darwin-10.9-amd64.tgz
        ~~~

        {% include copy-clipboard.html %}
        ~~~ shell
        # Extract the binary:
        $ tar xfz cockroach-{{page.release_info.version}}.darwin-10.9-amd64.tgz
        ~~~

        {% include copy-clipboard.html %}
        ~~~ shell
        # Optional: Place cockroach in your $PATH
        $ cp -i cockroach-{{page.release_info.version}}.darwin-10.9-amd64/cockroach /usr/local/bin/cockroach
        ~~~
    - **Linux**:

        {% include copy-clipboard.html %}
        ~~~ shell
        # Get the CockroachDB tarball:
        $ wget https://binaries.cockroachdb.com/cockroach-{{page.release_info.version}}.linux-amd64.tgz
        ~~~

        {% include copy-clipboard.html %}
        ~~~ shell
        # Extract the binary:
        $ tar xfz cockroach-{{page.release_info.version}}.linux-amd64.tgz
        ~~~

        {% include copy-clipboard.html %}
        ~~~ shell
        # Optional: Place cockroach in your $PATH
        $ cp -i cockroach-{{page.release_info.version}}.linux-amd64/cockroach /usr/local/bin/cockroach
        ~~~

3. Stop the `cockroach` process.

    Without a process manager, use this command:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ pkill cockroach
    ~~~

    Then verify that the process has stopped:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ ps aux | grep cockroach
    ~~~

    Alternately, you can check the node's logs for the message `server drained and shutdown completed`.

4. If you use `cockroach` in your `$PATH`, rename the outdated `cockroach` binary, and then move the new one into its place:
    - **Mac**:

        {% include copy-clipboard.html %}
        ~~~ shell
        $ i="$(which cockroach)"; mv "$i" "$i"_old
        ~~~

        {% include copy-clipboard.html %}
        ~~~ shell
        $ cp -i cockroach-{{page.release_info.version}}.darwin-10.9-amd64/cockroach /usr/local/bin/cockroach
        ~~~
    - **Linux**:

        {% include copy-clipboard.html %}
        ~~~ shell
        $ i="$(which cockroach)"; mv "$i" "$i"_old
        ~~~

        {% include copy-clipboard.html %}
        ~~~ shell
        $ cp -i cockroach-{{page.release_info.version}}.linux-amd64/cockroach /usr/local/bin/cockroach
        ~~~

    If you leave versioned binaries on your servers, you don't need to do anything.

5. If you're running with a process manager, have the node rejoin the cluster by starting it.

    Without a process manager, use this command:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ cockroach start --join=[IP address of any other node] [other flags]
    ~~~
    `[other flags]` includes any flags you [use to a start node](start-a-node.html), such as it `--host`.

6. Verify the node has rejoined the cluster through its output to `stdout` or through the [admin UI](explore-the-admin-ui.html).

7. If you use `cockroach` in your `$PATH`, you can remove the old binary:

    {% include copy-clipboard.html %}
    ~~~ shell
    $ rm /usr/local/bin/cockroach_old
    ~~~

8. Wait at least one minute after the node has rejoined the cluster, and then repeat these steps for the next node.

## Step 3. Monitor the upgraded cluster

After upgrading all nodes in the cluster, monitor the cluster's stability and performance for at least one day.

{{site.data.alerts.callout_danger}}During this phase, avoid using any new 1.1 features. Doing so will prevent you from being able to perform a rolling downgrade to 1.0, if necessary.{{site.data.alerts.end}}

## Step 4. Finalize or revert the upgrade

Once you have monitored the upgraded cluster for at least one day:

- If you are satisfied with the new version, complete the steps under [Finalize the upgrade](#finalize-the-upgrade).

- If you are experiencing problems, follow the steps under [Revert the upgrade](#revert-the-upgrade).

### Finalize the upgrade

{{site.data.alerts.callout_info}}These final steps are required after upgrading from v1.0.x to v1.1. For upgrades within the 1.1.x series, you don't need to take any further action.{{site.data.alerts.end}}

1. [Back up the cluster](back-up-data.html).

2. Start the [`cockroach sql`](use-the-built-in-sql-client.html) shell against any node in the cluster and execute the following query:

    ~~~ sql
    > SET CLUSTER SETTING version = '1.1';
    ~~~

    This step enables certain performance improvements and bug fixes that were introduced in v1.1. Note, however, that after completing this step, it will no longer be possible to perform a rolling downgrade to v1.0. In the event of a catastrophic failure or corruption due to usage of new features requiring v1.1, the only option is to start a new cluster using the old binary and then restore from one of the backups created prior to finalizing the upgrade.

### Revert the upgrade

1. Run the [`cockroach debug zip`](debug-zip.html) command against any node in the cluster to capture your cluster's state.

2. [Reach out for support](support-resources.html) from Cockroach Labs, sharing your debug zip.

3. If necessary, downgrade the cluster by repeating the [rolling upgrade process](#step-2-perform-the-rolling-upgrade), but this time switching each node back to the previous version.

## See Also

- [View Node Details](view-node-details.html)
- [Collect Debug Information](debug-zip.html)
- [View Version Details](view-version-details.html)
- [Release notes for our latest version](../releases/{{page.release_info.version}}.html)
