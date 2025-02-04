:desc: Read more about how to format training data with Rasa NLU for open
       source natural language processing.

.. _training-data-format:

Training Data Format
====================

.. edit-link::

.. contents::
   :local:

Data Formats
~~~~~~~~~~~~


You can provide training data as Markdown or as JSON, as a single file or as a directory containing multiple files.
Note that Markdown is usually easier to work with.


Markdown Format
---------------

Markdown is the easiest Rasa NLU format for humans to read and write.
Examples are listed using the unordered list syntax, e.g. minus ``-``, asterisk ``*``, or plus ``+``.
Examples are grouped by intent, and entities are annotated as Markdown links,
e.g. ``[<entity text>](<entity name>)``, or by using the following syntax ``[<entity-text>]{"entity": "<entity name>"}``.
Using the latter syntax, you can also assign synonyms, roles, or groups to an entity, e.g.
``[<entity-text>]{"entity": "<entity name>", "role": "<role name>", "group": "<group name>", "value": "<entity synonym>"}``.
The keywords ``role``, ``group``, and ``value`` are optional in this notation.
To understand what the labels ``role`` and ``group`` are for, see section :ref:`entities-roles-groups`.

.. code-block:: md

    ## intent:check_balance
    - what is my balance <!-- no entity -->
    - how much do I have on my [savings](source_account) <!-- entity "source_account" has value "savings" -->
    - how much do I have on my [savings account]{"entity": "source_account", "value": "savings"} <!-- synonyms, method 1-->
    - Could I pay in [yen](currency)?  <!-- entity matched by lookup table -->

    ## intent:greet
    - hey
    - hello

    ## synonym:savings   <!-- synonyms, method 2 -->
    - pink pig

    ## regex:zipcode
    - [0-9]{5}

    ## lookup:additional_currencies  <!-- specify lookup tables in an external file -->
    path/to/currencies.txt

The training data for Rasa NLU is structured into different parts:

- common examples
- synonyms
- regex features and
- lookup tables

While common examples is the only part that is mandatory, including the others will help the NLU model
learn the domain with fewer examples and also help it be more confident of its predictions.

Synonyms will map extracted entities to the same name, for example mapping "my savings account" to simply "savings".
However, this only happens *after* the entities have been extracted, so you need to provide examples with the synonyms
present so that Rasa can learn to pick them up.

Lookup tables may be specified as plain text files containing newline-separated words or 
phrases. Upon loading the training data, these files are used to generate
case-insensitive regex patterns that are added to the regex features.

.. note::
    The common theme here is that common examples, regex features and lookup tables merely act as cues to the final NLU
    model by providing additional features to the machine learning algorithm during training. Therefore, it must not be
    assumed that having a single example would be enough for the model to robustly identify intents and/or entities
    across all variants of that example.

.. note::
    ``/`` symbol is reserved as a delimiter to separate retrieval intents from response text identifiers. Make sure not
    to use it in the name of your intents.

.. warning::
    The synonym format to specify synonyms ``[savings account](source_account:savings)`` is deprecated. Please use the
    new format ``[savings account]{"entity": "source_account", "value": "savings"}``. To update your training data
    file execute the following command on the terminal of your choice:
    ``sed -i .deprecated -E 's/\[(.+)\]\((.+):(.+)\)/\[\1\]\{"entity": "\2", "value": "\3"\}/g' <nlu training data file>``
    Assume you training data file is called ``nlu.md``. After you executed the above command, this file will
    contain the new training data format. The command also creates a backup file with the old training data format,
    called ``nlu.md.deprecated``.

JSON Format
-----------

The JSON format consists of a top-level object called ``rasa_nlu_data``, with the keys
``common_examples``, ``entity_synonyms`` and ``regex_features``.
The most important one is ``common_examples``.

.. code-block:: json

    {
        "rasa_nlu_data": {
            "common_examples": [],
            "regex_features" : [],
            "lookup_tables"  : [],
            "entity_synonyms": []
        }
    }

The ``common_examples`` are used to train your model. You should put all of your training
examples in the ``common_examples`` array.
Regex features are a tool to help the classifier detect entities or intents and improve the performance.


Improving Intent Classification and Entity Recognition
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Common Examples
---------------

Common examples have three components: ``text``, ``intent`` and ``entities``. The first two are strings while the last
one is an array.

 - The *text* is the user message [required]
 - The *intent* is the intent that should be associated with the text [optional]
 - The *entities* are specific parts of the text which need to be identified [optional]

Entities are specified with a ``start`` and an ``end`` value, which together make a range
to apply to the string, e.g. in the example below, with ``text="show me chinese restaurants"``, then
``text[8:15] == 'chinese'``. Entities can span multiple words, and in
fact the ``value`` field does not have to correspond exactly to the substring in your example.
That way you can map synonyms, or misspellings, to the same ``value``.

.. code-block:: md

    ## intent:restaurant_search
    - show me [chinese](cuisine) restaurants


Regular Expression Features
---------------------------
Regular expressions can be used to support the intent classification and entity extraction. For example, if your entity
has a deterministic structure (like a zipcode or an email address), you can use a regular expression to ease detection
of that entity. For the zipcode example it might look like this:

.. code-block:: md

    ## regex:zipcode
    - [0-9]{5}

    ## regex:greet
    - hey[^\\s]*

The name doesn't define the entity nor the intent, it is just a human readable description for you to remember what
this regex is used for and is the title of the corresponding pattern feature. As you can see in the above example, you
can also use the regex features to improve the intent
classification performance.

Try to create your regular expressions in a way that they match as few words as possible. E.g. using ``hey[^\s]*``
instead of ``hey.*``, as the later one might match the whole message whereas the first one only matches a single word.

Regex features for entity extraction are currently only supported by the ``CRFEntityExtractor`` component! Hence, other
entity extractors, like ``MitieEntityExtractor`` or ``SpacyEntityExtractor`` won't use the generated features and their
presence will not improve entity recognition for these extractors. Currently, all intent classifiers make use of
available regex features.

.. note::
    Regex features don't define entities nor intents! They simply provide patterns to help the classifier
    recognize entities and related intents. Hence, you still need to provide intent & entity examples as part of your
    training data!

.. _lookup-tables:

Lookup Tables
-------------
Lookup tables provide a convenient way to supply a list of entity examples.
The supplied lookup table files must be in a newline-delimited format.
For example, ``data/test/lookup_tables/plates.txt`` may contain:

.. literalinclude:: ../../data/test/lookup_tables/plates.txt

And can be loaded and used as shown here:

.. code-block:: md

    ## lookup:plates
    data/test/lookup_tables/plates.txt

    ## intent:food_request
    - I'd like beef [tacos](plates) and a [burrito](plates)
    - How about some [mapo tofu](plates)

When lookup tables are supplied in training data, the contents are combined
into a large, case-insensitive regex pattern that looks for exact matches in
the training examples. These regexes match over multiple tokens, so
``lettuce wrap`` would match ``get me a lettuce wrap ASAP`` as ``[0 0 0 1 1 0]``.
These regexes are processed identically to the regular regex patterns
directly specified in the training data.

.. note::
    For lookup tables to be effective, there must be a few examples of matches in your training data. Otherwise the
    model will not learn to use the lookup table match features.


.. warning::
    You have to be careful when you add data to the lookup table.
    For example if there are false positives or other noise in the table,
    this can hurt performance. So make sure your lookup tables contain
    clean data.


Normalizing Data
~~~~~~~~~~~~~~~~

.. _entity_synonyms:

Entity Synonyms
---------------
If you define entities as having the same value they will be treated as synonyms. Here is an example of that:

.. code-block:: md

    ## intent:search
    - in the center of [NYC]{"entity": "city", "value": "New York City")
    - in the centre of [New York City](city)


As you can see, the entity ``city`` has the value ``New York City`` in both examples, even though the text in the first
example states ``NYC``. By defining the value attribute to be different from the value found in the text between start
and end index of the entity, you can define a synonym. Whenever the same text will be found, the value will use the
synonym instead of the actual text in the message.

To use the synonyms defined in your training data, you need to make sure the pipeline contains the
``EntitySynonymMapper`` component (see :ref:`components`).

Alternatively, you can add an "entity_synonyms" array to define several synonyms to one entity value. Here is an
example of that:

.. code-block:: md

    ## synonym:New York City
    - NYC
    - nyc
    - the big apple

.. note::
    Please note that adding synonyms using the above format does not improve the model's classification of those entities.
    **Entities must be properly classified before they can be replaced with the synonym value.**
