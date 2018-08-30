Mega Fish
=========
A generator for content channels trees in "channel fixtures json" format used
for tests throughout the Kolibri content pipeline: `Ricecooker --> Studio --> Kolibri`


Use cases
---------
  1. Create an in-memory tree built from `ricecooker` node class instances
  2. Create a sample ricecooker_json_tree.json to test json_chef workflow
     (check if resuliting in-memory tree matches structure in 1.)
  3. Generate sample "ricecooker wire format" json data for testing internal API
     endpoints (check if metadata )




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
        objects = TreeManager()
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

        # Legacy fields
        cloned_source = TreeForeignKey('self', on_delete=models.SET_NULL, null=True, blank=True, related_name='clones')
        original_node = TreeForeignKey('self', on_delete=models.SET_NULL, null=True, blank=True, related_name='duplicates')



# the content_id is used for tracking a user's interaction with a piece of
# content, in the face of possibly many copies of that content. When a user
# interacts with a piece of content, all substantially similar pieces of
# content should be marked as such as well. We track these "substantially
# similar" types of content by having them have the same content_id.
