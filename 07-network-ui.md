# Network visualization

Once the `hlom_citation` table is filled in with syllabus -> text associations, we can render out the network visualization UI.

## Construct the graph

1. First, we need to write "deduping" hashes onto the HLOM records table. This will make it possible to eliminiate duplicate nodes in the network (eg, where Harvard has 20-30 different variations of "Republic, Plato"). Easiest to do this in an IPython terminal:

```
ipython3
> from osp.citations.hlom.models.record import HLOM_Record
> HLOM_Record.dedupe()
```

1. Then, still in the IPython shell, make an instance of the `Network` class:

```
from osp.citations.hlom.network import Network
n = Network()
```

1. Fill in the edges list:

```
n.add_edges()
```

1. Scrub out duplicate texts:

```
n.dedupe()
```

1. Delete "unconnected components," isolated subgraphs that aren't connected to the largest connected graph:

```
n.trim_unconnected_components()
```

(It may actually be interesting to look at these unconnceted subgraphs and see what they are - for the purposes of the UI, though, we need to remove them, because they will cause trouble when it comes time to run the force-directed layout in Gephi - they will get shot off really far away from the main body of the graph, which will mess up the calculations that try to center the main body of the network when the big image is rendered.)

1. Last, save off the network as a .gexf file:

```
n.write_gexf('/path/to/network.gexf')
```

## Run the layout in Gephi

1. Open up Gephi, and load the .gexf file with File > Open. For **Graph Type**, select "Undirected," and then hit "OK."

1. At the bottom left of the screen, under **Layout**, select "Force Atlas 2" from the dropdown, and bump up the **Threads** count to 4.

1. Click **Run** to start the layout algorithm. Let this run for a while, at least 4-5 hours. These alrogithms are basically physical simulations, and don't have an end state, but they do settle into an equilibrium after which more iterations won't do much to change the structure.

1. Once the layout is done, click the **Run** button next to "Modularity" on the right side of the screen.

1. Once this finishes, find the bock at the top left of the screen, with tabs for **Partition** and **Ranking**. Click on **Partition**, and then select the **Nodes** tab just below it, and click the little "refresh" button with the green arrows. This should fill in the dropdown box - select "Modularity Class," and then click **Run** to apply the component colors.

1. Last, we can size the nodes according to "degree" the number of edges that are connected to the node. Click on the **Ranking** tab at the top right, and then select the little red "diamond" icon, which, for some odd reason, represents size. Select "Degree" from the drop-down list, and click **Apply** to render the sizes. Fiddle a bit with the **Min Size** and **Max Size** options to find a range of sizes that looks good.

1. Last, once you like the color and size settings, flip on the "Prevent Overlap" option in the layout settings, and let the layout run for another 10-15 minutes. This allows Gephi to reposition the nodes so that they don't overlap with one another (as much as possible). Be sure to do this any time you make a change to the node sizes, or else you'll get weird gaps or overlaps in the final image render.

1. Once everything is done, save off a new version of the graph via File > Export > Graph File. Select "GEXF Files" in the dropdown, and give it a new name to keep it separate from the original file that came out of the OSP code.

## Render the image

Usually, we could just use Gephi to render out a big, high-quality version of the image. But, in this case, because there are so many nodes, we need to draw out a enormous, ~200k X 200k image, which Gephi can't handle.

1. First, spin up an instance on EC2 with a _huge_ amount of RAM - I had good experiences with the r3.4xlarge. These are expensive, but we'll only need to leave it up for 1-2 hours, so it will just cost about $5.

1. SCP up the .gexf file that came out of Gephi. Start a new tmux session, since the next steps will take a long time. Open an IPython shell and make an instance of GephiNetwork:

```
from osp.citations.hlom.network import GephiNetwork
n = GephiNetwork.from_gexf('/path/to/file.gexf')
```

1. First, we need to pull in the authors and titles for the nodes. This needs to be done after running the layout in Gephi, because Gephi has weird encoding problems that will flatten out non-ASCII characters. Do this with:

```
n.hydrate_nodes()
```

1. Then, start the image render:

```
n.render('osp.png', scale=6, size=220000)
```

The "scale" is basically a magnification factor - if it's 6, then the final image will be 6 times larger than the original coordinate values that came out of Gephi. We need to do this in order to make the image large enough that the text labels are readable. The "size" is just the height and width of the image. You may need to fiddle with these parameters - lots of things could change the underlying size of the network (the number of citations, the number of texts, etc), which could make it necessary to bump the size of the render up or down.

1. **Important** - Once the render finishes, don't close out of teh IPython shell. Only at this point do we have the data for the actual, "physical" coordinates of the nodes as they were positioned in the final image, and we need to save this data so that we can index it into Elasticsearch. Save off a third .gexf file, which will retain this data:

```
n.write_gexf('/path/to/file-with-positions.gefx')
```

## Slice the image

Next, we need to chop up the huge image into a tile pyramid, which can then be published to the web with the `osp-network` viewer.

1. On the rendering instance, install `vips`, an image processing utility:

```
sudo apt-get install libvips
```

1. As always - be sure to open up a tmux session, since this will take a long time. Assuming the image is called `osp.png`, slice the image with:

```
vips dzsave osp.png tiles --layout zoomify
```

This will take about an hour. When it's finished, there will be a `tiles` directory filled with about 4,000 sub-directories, each containing a group of the tiles in the pyramid.

1. Last, move these tiles onto the extraction volume, since we'll need them when we deploy a smaller instance for the web-facing interface. I've been putting the different iterations in `/osp/tiles/v{x}` directories, where `x` just gets bumped up every time I re-render the image. So, for a new set of tiles:

```
sudo cp -r tiles /osp/tiles/v4
```

1. At this point, be sure to snapshot the volume.

## Index Postgres and Elasticsearch

One more step before we can depoy a fresh version of the UI for the network - we need to index information about the nodes and edges in Postgres and Elasticsearch. The `osp-network` code needs this data to run the searching and nearest-neighbor queries.

1. On the render instance, open up an IPython shell and make an instance of `GephiNetwork` with the last version of the .gexf file that was saved off after the image was rendered:

```
from osp.citations.hlom.network import GephiNetwork
n = GephiNetwork.from_gexf('/path/to/most/recent/file.gexf')
```

1. First, create the Elasticsearch mapping, and then index the nodes:

```
n.es_create()
n.es_insert()
```

1. Then, instert nodes into Postgres:

```
n.pg_insert_nodes()
```

1. And last, edges into Postgres:

```
n.pg_insert_edges()
```

This last step will take a long time - to make the nearest-neighbors queries workably fast, we have to index each edge in both directions. Eg, if A is connected to B - we have to index rows for both A->B and B->A. Since there are ~3M edges, this means we have to write ~6M rows.

1. When this is finished, snapshot the extraction volume and terminate the render instance.

## Deploy the viewer

Now, we can deploy the pubic-facing viewer UI for the network.

1. Clone the `osp-network` repository. The Ansible rig is organized in almost exact same way as in the the `osp` project. Change down into the `deploy` folder, and open the secrets file:

```
ansible-vault edit group_vars/all/global.yml
```

And paste in the id of the snapshot with the tile pyramid and indexes nodes/edges that we created in the last step.

1. Open up the `vars/ec2.yml` file, and set an instance size that makes sense.

1. Then, run the three main playbooks to put up the server and deploy the code:

```
ansible-playbook provision.yml
./ec2/ec2.py --refresh-cache
ansible-playbook configure.yml
ansible-playbook deploy.yml
```

1. Last - if the nearest-neighbors queries are sluggish, throw a descending sort index on the `weight` column on the `hlom_edge` table:

```
psql osp -U osp
> CREATE INDEX ON hlom_edge (weight desc);
```
