Mega Fish
=========
A generator for content channels trees in "channel fixtures json" format used
for tests throughout the Kolibri content pipeline: `Ricecooker --> Studio --> Kolibri`

Goals
-----
  - Test all content kinds supported:
    - Audio (including thumbnail generation)
    - Video (including compression and thumbnail generation)
      - video conversion
      - subtitles and subtitle conversion
    - Document
      - pdf
      - ePub
    - HTML5App

  - Test exercises
    - Ricecooker must correctly math,  md image includes, and graphie urls images
    - Studio must correctly create .perseus zip from ContentNode, AssesmentItems, and Files
    - Kolibri import should load .perseus zip files and have matching pointers in the DB
    - Kolibri should be able to render .perseus zip files

  - Load testing the ecosystem with large trees
    - Tree generator for load testing Kolibri
    - Tree generator for load testing Studio

  - Test with randomly generated trees?



Use cases
---------
  1. Create an in-memory tree built from `ricecooker` node class instances
  2. Create a sample ricecooker_json_tree.json to test json_chef workflow
     (check if resuliting in-memory tree matches structure in 1.)
  3. Generate sample "ricecooker wire format" json data for testing Studio
     Internal API endpoints (check if all metadata is stored and handled correctly)
     - Generating large channels for stress testing Internal API
  4. Test the Studio exportchannel functionality
  5. Test Koilibri importchannel functionality



Studio content nodes data model
-------------------------------
The main source of truth for all things content-related in the Kolibri ecosystem
is the Kolibri Studio database, accessed through the Django ORM via this model:

    class ContentNode(MPTTModel, models.Model):
        """
        Studio ContentNode = elements of the trees associated with content channels
        """
        # One of ['topic', 'audio', 'document', 'audio', 'video', 'html5', 'epub'
        kind = models.ForeignKey('ContentKind', related_name='contentnodes', db_index=True)

        # Random id used internally on Studio (See `node_id` for id used in Kolibri)
        id = UUIDField(primary_key=True, default=uuid.uuid4)

        # Tree structure
        parent = TreeForeignKey('self', null=True, blank=True, related_name='children', db_index=True)
        # sort order of node within parent node's children
        sort_order = models.FloatField(max_length=50, default=1, verbose_name=_("sort order"),

        # Ids shared between Ricecooker, Studio, and Kolibri
        source_domain = models.CharField(max_length=300, blank=True, null=True)
        source_id = models.CharField(max_length=200, blank=True, null=True)
        content_id = UUIDField(primary_key=False, default=uuid.uuid4, editable=False, db_index=True)
        node_id = UUIDField(primary_key=False, default=uuid.uuid4, editable=False)

        # Store provenance links for content nodes remixed through Studio web UI
        # original_* ids refer to original node at the beginning of provenance chain
        original_channel_id = UUIDField(primary_key=False, editable=False, null=True, db_index=True)
        original_source_node_id = UUIDField(primary_key=False, editable=False, null=True, db_index=True)
        # source_* ids refer to previous channel in provenance chain
        source_channel_id = UUIDField(primary_key=False, editable=False, null=True)
        source_node_id = UUIDField(primary_key=False, editable=False, null=True)  # Immediate node_id of node copied from

If we generate "fixtures" for Studio, we can cover the other parts of the pipeline.

For a more detailed table of the data model at all steps of the content pipeline,
see [this doc](https://docs.google.com/spreadsheets/d/181hSEwJ7yVmMh7LEwaHENqQetYSsbSDwybHTO_0zZM0/edit#gid=1640972430).



IDs
---
It's worth 

  - `channel_id`: is the main identifier within Kolibri content ecosystem	that 
    you need to know about.
    ```
    channel_id = uuid.uuid5(uuid.uuid5(uuid.NAMESPACE_DNS, source_domain), source_id).hex
    ```
    where `source_domain` (str) is the Website domain of content producer, and
    `source_id` (str) is the upsream ID for channel within the source domain.

On ricecooker and Kolibri channels are one-to-one with one tree, a.k.a "main" tree.
On Studio channels can have the following trees:
  - `main_tree`: the current state of the channel. This is the tree the is used to generate the versioned Kolibri sqlite3 DB on `PUBLISH`.
  - `staging_tree`: this is the "draft version" for the main tree that is used as preview. The `DEPLOY` action moves the staged tree to the main tree.
  - `chef_tree`: tree of ContentNode objects as build during a ricecooker chef run
  - `previous_tree`: a "backup copy" of the main tree (created before `DEPLOY`)
  - `trash_tree` ?? 
  - clipboard_tree (legacy)


Content nodes come in many kinds specified by their `kind` (str) attribute.
Each ContentNode has the following IDs:
  - `node_id`: identifier within the channel
  - `source_id`: Upstream ID for this content item
  - `content_id`: Main content identifier within Kolibri ecosystem
    The content_id is used for tracking a user's interaction with a piece of
    content, in the face of possibly many copies of that content. When a user
    interacts with a piece of content, all substantially similar pieces of
    content should be marked as such as well. We track these "substantially
    similar" types of content by having them have the same content_id.
  - On Kolibri 
    - `tree_id`: Which tree is this node part of?
    - `get_channel()`: Which channel is this node part of?
  - `id` (a.k.a `studio_id`): is a randomly generated primary key
    on Studio that is used for the tree structure (parent+children FKs)
	-	`parent_id`: FK to Studio Internal ID of parent node


We can generate `node_id`, `source_id`, and `content_id` can all
be generated as part of fixtures, and we'll have to work with
the studio-generated `id` when doing ricecooker--studio tests.




Design
------
Use [`mixer`](https://github.com/klen/mixer#common-usage) to generate
random attributes and tie them together into a hierarchy using `pytest`.


    @pytest.fixture
    def contentnode_f(video_file):
        pass

    @pytest.fixture
    def contentnode_s(contentnode_f, videofile_f):
        pass



Links
-----
  - Mixer types: https://github.com/klen/mixer/blob/develop/mixer/factory.py#L61-L88
  - Test RTL languages too https://stackoverflow.com/a/42557555/127114
    using `mixer.faker.locale = 'ar'` and maybe also add dir-chars.
    quote: Put a Right-to-Left Embedding character, u'\u202B', at the beginning of each Hebrew word,
    and a Pop Directional Formatting character, u'\u202C', at the end of each word.
    OR u'\u2067' RIGHT-TO-LEFT ISOLATE and u'\u2069' POP DIRECTIONAL ISOLATE, not RIGHT-TO-LEFT EMBEDDING and POP DIRECTIONAL FORMATTING.
    A right-to-left directional embedding has roughly the effect of a strong RTL character on the surrounding text, 
    which can cause undesirable reordering of text you wanted to isolate from the effects of the RTL characters inside the embedding.


