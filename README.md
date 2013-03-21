# Puppet catalog test

This is a simple set of scripts that can be use to compile the catalogs of your different nodes with different puppet versions. The catalogs will be stored for later analysis, for example using `puppet catalog diff` to see if catalogs containing the same resources with the same settings are produced with different puppet versions.

This will allow you to safely compile all your catalogs of your nodes with a new puppet version *before* rolling out that version on your master. Also you might detect deprecations, incompatibility and so on already on your local node/CI server/wherever you run it.

## Requirements

1. bundler - to install/load the requirements and safely switch between the puppet versions.
1. The facts-yaml files of all your nodes from the master in `/var/lib/puppet/yaml`. The easiest way to gather these files is to take a tarball of your `/var/lib/puppet/yaml` directory on your master and extract it in `var/yaml` of this respository. This is required to compile the catalogs of your nodes with the same facts, as the nodes recently reported on the master.
1. your puppet manifests in `../puppet` - or set PUPPETDIR in `compile` accordingly.
1. your hieradata (if you use yaml hieradata) somewhere and a copy of your hiera.yaml configuration file from your master in `etc/` with the yaml-source-path pointing to your local hiera-yaml-store. Same goes for things like trocla and other sources for your manifests.
1. Configure `--modulepath` in `compile` according to your setup and also any other option that might be different than the provided ones. If you are using an ENC, you will also need to configure it by either setting it up in `etc/puppet.conf` or as a cmd-line parameter in `compile`.
1. puppet-catalog-diff - Active your puppet >= 3 installation and simply run `puppet module install ripienaar/catalog_diff` to install it.
1. If you are using any special functions (like generate(), file() or name-your-internal-function) in your manifests, you need to ensure that they are also working in (a similar) way on the machine your are compiling the catalogs. This means for example that they can access the same filesystem paths as your puppet-master would. Often they don't need to contain the same content as the master - so no need to copy your private files on your client - but if for example the paths for file() are hardocded as full qualified paths in your manifests, your client will also need to read these files.

## Compiling catalogs

If you have setup everything and your facts are in place, compiling a catalog for a node should be as simple as:

    ./compile <PUPPETVERSION> <FQDN>
    ./compile 2.7 node.example.com

The last command will compile the catalog for the node node.example.com with the puppet version 2.7 (2.7.21 to be precise) and store the overall output in `puppet_catalogs_2.7/node.example.com.out`. It will also try to extract the compile time to `puppet_catalogs_2.7/node.example.com.time` and the catalog itself to `puppet_catalogs_2.7/node.example.com.pson`. These extraction commands are very basic and could definitely be more robust. But they worked for my setup. Problems might for example arise if the master gives more output than a simple one-liner for the compile time. So for example deprecation warnings and so on.

You might want to loop that compilation over all your nodes with a simple for-loop:

    for i in `ls -1 var/yaml/facts/*.yaml` ; do node=`basename $i .yaml`; ./compile 2.6 $node; done

and for all puppet versions and all puppet nodes:

    for version in 2.6 2.7 3; do for i in `ls -1 var/yaml/facts/*.yaml` ; do node=`basename $i .yaml`; ./compile $version $node; done; done

### Exported resources

As we are starting with an empty database in our temporary exported resources database, you might want to compile your overall infrastracture at least twice to get a full set of exported resources, so that the catalogs aren't changing anymore due to freshly exported resources by other hosts *before* diffing the catalogs with the different versions. This can happen with the same puppet version.
This will also make the compile time of the first puppet version probably more comparable to the ones with the other versions, as less things have to be stored on the second run.

## First checks for problems

A good thing to look for potential issues with a newer puppet version than your current is to look at the output of the compile command. STDERR is printed on your cli, while STDOUT goes completely to `puppet_catalogs_2.7/node.example.com.out`. So any deprecation warnings or other output might end up in the `.out` file. The catalog usually starts with a `{` and looks like a dumped hash in pretty printed JSON.

## Catalog diffing

After you compiled the catalogs of a node with at least two versions, you can start comparing them, using puppet-catalog-diff. Be aware that puppet-catalog-diff only works with puppet 3 as a face command, but can still analyze catalogs produced by older versions.

    puppet catalog diff puppet_catalogs_2.7/node.example.com.pson puppet_catalogs_3/node.example.com.pson

This will give you a diff for the 2 catalogs if there is any. Have a look at https://github.com/ripienaar/puppet-catalog-diff#readme for more information.

If you want the output to be less verbose and automatically diffing the catalogs of all versions going upwards, you can use the `diff_all` command. This will also extract the compile time from the different catalogs and display them in a sequence for easier comparison.

    for i in `ls -1 var/yaml/facts/*.yaml` ; do node=`basename $i .yaml`; ruby diff_all $node; done

Have fun!

# Who - License

duritong - GPLv2

## Excuse

These scripts are working for me(TM) and got only little cleanup before sharing with you. You might need to do some adjustments/fixing to make it work for you. I'm looking forward to pull requests.
