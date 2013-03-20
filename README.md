# Puppet catalog test

This is simple set of scripts that one can use to compile the catalogs of your different nodes with different puppet versions. The catalogs will be stored for later analysis, for example using `puppet catalog diff` to see if catalogs are the same on different versions.

This will allow you to safely compile all your catalogs of your nodes with a new puppet version *before* rolling out that version on your master. Also you might detect deprecations, incompatibility etc. already on your local node/CI server/wherever you run it.

## Requirements

1. bundler - to install/load the requirements and safely switch between the puppet versions.
1. facts-yaml of all your nodes from the master in var/yaml. Usually this means to just take a tarball of your /var/lib/puppet/yaml directory on your master and extract it in `var/yaml` in this directory. This is required so that the catalogs for your node are compiled with the same facts as they reported recently on the master.
1. your puppet manifests in `../puppet` - or set PUPPETDIR in `compile` accordingly.
1. your hieradata (if you use yaml hieradata) somewhere and a copy of your hiera.yaml configuration file from your master in `etc/` with the yaml-source-path adjusted. Same goes for things like trocla and so on.
1. adjusting `--modulepath` in `compile` to fit your needs and also any other option that might be different than the provided ones. You might also need to configure your ENC, either in `etc/puppet.conf` or as cmd-line parameter in `compile` if you use one.
1. puppet-catalog-diff - While having puppet >= 3 activated just run `puppet module install ripienaar/catalog_diff` to install it.
1. If you are using any special functions (like generate(), file() or name-your-internal-function) in your manifests, you need to ensure that they are also working in (a similar) way on the machine your are compiling the catalogs.

## Compiling catalogs

If you have setup everything and your facts are in place, compiling a catalog for a node should be as simple as:

    ./compile <PUPPETVERSION> <FQDN>
    ./compile 2.7 node.example.com

The last command will compile the catalog for the node node.example.com with the puppet version 2.7 (2.7.21 to be precise) and store the overall output in `puppet_catalogs_2.7/node.example.com.out`. It will also try to extract the compile time to `puppet_catalogs_2.7/node.example.com.time` and the catalog itself to `puppet_catalogs_2.7/node.example.com.pson`. These extraction commands are very rudimentary and might require adjustments if your master gives more output than a simple one-liner about the compile time.

You might want to loop that compilation over all your nodes with a simple for-loop:

    for i in `ls -1 var/yaml/facts/*.yaml` ; do node=`basename $i .yaml`; ./compile 2.6 $node; done

and for all puppet versions and all puppet nodes:

    for version in 2.6 2.7 3; do for i in `ls -1 var/yaml/facts/*.yaml` ; do node=`basename $i .yaml`; ./compile $version $node; done; done

### Exported resources

As we are starting with an empty database in our temporary exported resources database, you might want to compile your overall infrastracture at least twice to get a full set of exported resources, so that the catalogs aren't changing anymore due to freshly exported resources by other hosts *before* diffing the catalogs with the different versions. This can happen with the same puppet version.
This will also make the compile time of the first puppet version probably more comparable, as less things have to be stored on the second run.

## First checks for problems

A good thing to look for potential issues with a newer puppet version than your current is to look at the output of the compile command. STDERR is printed on your cli, while STDOUT goes completely to `puppet_catalogs_2.7/node.example.com.out`. So any deprecation warnings or other output might endup there and can influence the outcome of the extraction commands.

## Catalog diffing

After you compiled the catalogs of a node with at least two versions, you can start comparing them, using puppet-catalog-diff, when having puppet 3 activated:

    puppet catalog diff puppet_catalogs_2.7/node.example.com.pson puppet_catalogs_3/node.example.com.pson

This will give you a short overview if there are any differences at all.

If you want the output to be less verbose and automatically diffing the catalogs of all versions going upwards, you can use the `diff_all` command. This will also extract the compile time from the different catalogs and display them in a sequence for easier comparasion.

    for i in `ls -1 var/yaml/facts/*.yaml` ; do node=`basename $i .yaml`; ruby diff_all $node; done

Have fun!

# Who - License

duritong - GPLv2

## Excuse

These scripts are working for me and got little polishment to share with you. You might need to do some adjustments/fixing to make it work for you. I'm looking forward to pull requests.
