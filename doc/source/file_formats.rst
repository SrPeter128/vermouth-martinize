File formats
============

Vermouth has two main types of data files:

* Force fields describe :ref:`blocks <data:block>`, :ref:`links <data:link>` and :ref:`modifications <data:modification>` to generate
topologies.

* :ref:`Mappings <data:mapping>` describe the transformations
  required for going from one description (force field) to another or
  vice versa.

The two types of data are contained in data/force_fields and
data/mappings, respectively. The force_fields directory has a
subdirectory for each force field that is available in vermouth and
martinize2. The mappings directory has no mandatory organization; any
mapping applies to two force fields, which are specified in the
mapping file. For convenience, the mappings may be organized in
subfolders. In particular, mappings from and to the canonical
description, which is based on the charmm36 force field, are typically
placed in a subdirectory with the name of the other force field (e.g.,
martini30).

Data structures and file formats
--------------------------------

Please note that the file formats described here may undergo changes
in the future. Features that are in the code, but are not described
here should not be considered stable and will likely be deprecated in
the future.

For its description of the force fields, topologies and mappings,
Vermouth uses a file format based on the one used by Gromacs,
consisting of named sections and subsections, each indicated with a
tag between square brackets. Tags are divided into top-level and
sub-level section names.

Force field file (.ff)
---------------------------

The top-level tags for the force field and the topologies are macros,
variables, citations, moleculetypes, links, and modifications.

Macros
~~~~~~

The macros section has no further subsections and lists substitution
patterns to be applied throughout the file being read. Macro values
are substituted using the name with a preceding $. This is similar to
the use of variables in shell scripting and makes it easy to write
generalizations and to use and change default values. An example of
this section is

[ macros ]
prot_default_bb_type P2
stiff_fc 1000000

Variables
~~~~~~~~~

The variables section has no further subsections and lists control
parameters for the force field. An example of this is

[ variables ]
elastic_network_bond_type 1
res_min_dist 3

Citations
~~~~~~~~~

The citations section lists the citations to be used for the force
field. Citations are named and refer to an entry in the bibtex file
`citations.bib` in the force field directory. An example of this is

[ citations ]
Martini3

Citations can also be specified for moleculetypes, links, and
modifications, in a citation subsection.

Moleculetypes
~~~~~~~~~~~~~

The moleculetype describes a :ref:`block <data:block>`, i.e., a
topological building block, comprising of particles (atoms) with their
properties and interactions. This can be a separate molecule or a part
of a larger molecule, typically a monomeric unit in a polymer. The
moleculetype has a name and the number of bonds to use for exclusions
as its first content line. This is followed by one or more
subsections. The subsection atoms is mandatory, optional subsections
are edges and subsections corresponding to the different types of
atomic interactions, including bonds, constraints, angles, and
dihedrals.

Subsection metadata
^^^^^^^^^^^^^^^^^^^

Subsections can be given metadata, using a #meta directive. These
directives always apply to the whole subsection. This makes it
possible to, e.g., group interactions or annotate bonds according to
their type. It is also possible to specify a context for the
interactions in a subsection, using 'ifdef' and/or 'ifndef' tags. This
will make the inclusion of the interactions conditional by adding
#ifdef and #ifndef directive statements to the output topology
files. The metadata is given as a JSON style mapping of key/value
pairs.

#meta {"group": "Side chain bonds", "ifdef": "FLEXIBLE"}

Metadata can also be added to a single line by adding an attribute
statement as the last element.

Atoms
^^^^^

Each line in the atoms section describes one particle, corresponding
to a node in the molecular graph. The description comprises the atom
name, atom type, residue identifier, mass and charge.

Edges
^^^^^

Edges will be added to the molecular graph when required through
interactions, but they can also be added explicitly by listing them
under the edges subsections, specifying each edge to be added by the
corresponding atom names.

Interactions
^^^^^^^^^^^^

There are several options for subsections describing interactions
between particles. Of these, bonds, angles, dihedrals, cmap, and
constraints will automatically add the corresponding edges to the
molecular graph, unless specified explicitly by setting an attribute
'edge' to false in a subsection #meta or following a specific
interaction.

Each line in an interactions subsection specifies one interaction by
listing the atoms involved by name, followed by the interaction
parameters. For all interactions, the parameters are read as is and
written to the output topology without interpreting and/or
checking. Bond/constraint lengths, angles and dihedral angles may be
used for generating missing coordinates.

A full list of interactions is given below, corresponding to the list
of intramolecular interactions available in Gromacs, with a number
specifying the number of particles involved in the interaction. Note
that improper dihedrals are listed as a separate interaction type,
whereas in Gromacs these fall under the dihedrals section.

* bonds(2)
* angles(3)
* dihedrals(4)
* impropers(4)
* constraints(2)
* pairs(2)
* pairs_nb(2)
* SETTLE(1)
* virtual_sites2(3)
* virtual_sites3(4)
* virtual_sites4(5)
* position_restraints(1)
* distance_restraints(2)
* dihedral_restraints(4)
* orientation_restraints(2)
* angle_restraints(4)
* angle_restraints_z(2)
* cmap(...)

Any of the subsections can be given multiply times, in which case they
are additive. Do note that in the output topology specifying the same
interaction several times (the same type and particles) will overrule
any previous one, except when they are given different contexts (see
below).

Links
~~~~~

To generate a topology for a polymer or any molecule constructed from
joining parts, Vermouth connects moleculetypes using links. A link
describes how blocks are to be joined, what changes are effected in
the atom lists and which interactions are added, removed, or
altered. The changes in the atom and interaction lists are specified
using the corresponding subsections as under moleculetype. Further
subsections available for links are patterns, features, molmeta,
edges, and non-edges

The lines following the section tag may list selection statements for
filtering atoms in which to search for matching patterns. Each line
specifies a property and the corresponding value. The selection
statements may include filters based on, e.g., the residue name and
the secondary structure type, which are used to determine the
structural properties of protein backbone in the Martini force field.

Note: the link should have a name as overall content, to be consistent
with moleculetype and modification (and to increase clarity), and the
filters should probably fall under a subsection [ filters ].

Patterns
^^^^^^^^

Patterns is a subsection that lists patterns of atoms to which the
link applies. Each line in the subsection describes a pattern. One of
the patterns must apply for the link to match. A pattern consists of
atom identifiers. Each atom identifier consists of a name which may be
preceded by a shift and which may be further specified using an
attributes statement. The shift indicates a relative position in terms
of residues:

* +, ++, +++ : 	first, second, third following residue
* -, --, --- : 	first, second, third previous residue
* > :		other residue (not in order)

Thus, +CA in amino acids refers to the C-alpha atom in the C-terminal
connected neighbor, while >SG in the construction of a disulphide
bridge will refer to the SG atom in the partner cysteine.

An attributes statement is a JSON style mapping of key/value pairs,
similar to those described above (see #meta). For a pattern to match,
the atom names and the attributes need to match.

Shift operators can also be used in the interaction subsections.

Features
^^^^^^^^

The features subsection lists features to apply to the link
itself. These can be used to control the application of links during
the building of topologies. For example, setting the feature 'scfix'
will cause the links to be applied only if the option -scfix is given
to martinize2.

Molmeta
^^^^^^^

The molmeta subsection lists metadata to be added/changed in the
molecular graph. These metadata can be used (and modified) by
Vermouth's processors and for provenance.

Edges and non-edges
^^^^^^^^^^^^^^^^^^^

Within the context of links, the edges and non-edges specify that an
edge should or shouldn't be present, respectively, for the link
pattern to match. This behavior will likely be deprecated in the near
future in favor of filter keywords 'edge' and 'nonedge'. The current
behavior is really weird, because there is filtering and there are
patterns to do matching, and then edges and non-edges also filter?
Intuitively, edges and non-edges should specify the state after
application of the link, similar to the other interactions. I would
think that selection properties 'edge' and 'nonedge' in the filtering
would be more intuitive and useful.

Modifications
^^^^^^^^^^^^^

Modifications can be used to edit molecules or parts thereof (blocks),
e.g., for specifying protonation states. Each modification starts with
a line with the name. Thereafter may follow subsections as under
links. A modification may add, remove, or change atoms, interactions
and/or edges, using the corresponding subsections. Subsections
patterns, features, molmeta, edges, and non-edges function as under
links.

Atoms
^^^^^

The atoms subsection under a modification lists both anchors and atoms
to be added to anchors or changed. Entries consist of an atom name
followed by an attributes statement. Atoms that are added need to set
the "PTM_atom" attribute to true and require a valid "element"
attribute. The "replace" attribute may be set to a (nested) JSON dict,
listing the atom attributes to be changed and the new values
corresponding to the modification. Such changes can also be applied to
atoms already present in the molecular graph, i.e., the 'non-PTM
atoms'.`

Interactions
^^^^^^^^^^^^

Interactions are added or changed under the same interaction
subsections as used for moleculetype. How are interactions removed?

Mapping Files (.map & .mapping)
-----------------------------------------------
A mapping specifies the conversion from one force field description to
another. If the transformation is from a higher resolution force field
to a lower resolution, e.g., from the canonical description to
Martini, the process is typically called 'forward mapping'.

The vermouth library currently utilizes two mapping formats. The ``.map``
format, which was originally developed for the backward program, is
used to describe how two :ref:`blocks <data:block>` correspond to each
other. The second format (``.mapping``) is exclusively used in the
context of :ref:`modifications <data:modification>` and is an extension
to the first format.

File Structure (.map)
~~~~~~~~~~~~~~~~~~~~~

The file is structured into sections, each beginning with a directive
enclosed in square brackets (``[]``).

Allowed Directives
^^^^^^^^^^^^^^^^^^
The format recognizes the following directives:

- ``[molecule]``
    - This directive is immediately followed by a single line containing
      an alphanumeric string specifying the residue name. This name
      denotes the residue under consideration.
    - ``mandatory``

- ``[from]``
    - The directive is followed by a single line containing an
      alphanumeric string corresponding to name of the origin
      (i.e. higher resolution) force field.
    - ``mandatory``

- ``[to]``
    - The directive is followed by a single line containing an
      alphanumeric string corresponding to the name of the target
      (i.e. higher resolution) force field.
    - ``mandatory``

- ``[martini]``
    - The directive is followed by any number of lines. Each line must
      contain space separated bead names.
    - ``mandatory``
- ``[atoms]``
    - This directive introduces a section that can span multiple lines.
      Each line within this section must adhere to the following
      format:

      - An integer specifying the atom number.
      - An alphanumeric string corresponding to an atom name in the
        origin force field.
      - Any number of bead names. These beads must have been previously
        listed under the ``[martini]`` directive.
    - ``mandatory``
- ``[chiral]``
    - Contains chirality specifications used for the original backwards
      program.
    - ``ignored``
- ``[trans]``
    - Contains geometry specifications used for the original backwards
      program.
    - ``ignored``
- ``[out]``
    - Contains geometry specifications used in the original backwards
      program.
    - ``ignored``


Example
^^^^^^^

.. code-block:: plaintext

    [ molecule ]
    ALA ALA
    [ martini ]
    BB SC1
    [ atoms ]
     1     N    BB
     2    HN    BB
     3    CA    BB
     5    CB    SC1
     9     C    BB
    10     O    BB


File Structure (.mapping)
~~~~~~~~~~~~~~~~~~~~~
.. _mapping-file-format:
The file is structured into sections, each beginning with a directive
enclosed in square brackets (``[]``).

Allowed Directives
------------------

- ``[modification]``
    - Marks the beginning of a modification block.
      This directive does not require any following content.
    - ``mandatory``

- ``[from]``
    - Followed by the name of the origin force-field (e.g., amber).
    - ``mandatory``

- ``[to]``
    - Followed by the name of the target force-field (e.g., martini3001).
    - ``mandatory``

- ``[from blocks]`` and ``[to blocks]``
    - Each followed by an alphanumeric string describing a block name
      that needs to be modified by the modification at hand.
    - ``mandatory``

- ``[from nodes]``
    - Lists all nodes present in the modification description.
    - ``mandatory``

- ``[from edges]``
    - Contains all edges between the to-be-modified block and the
      modification.
    - ``mandatory``

- ``[mapping]``
    - Contains pairs of atom names and bead names, describing the
      actual mapping between the high-resolution and coarse-grained
      representations of the modification.

Example File
^^^^^^^^^^^^

Below is an example of a ``.mapping`` file:

.. code-block:: plaintext

    [modification]
    [from]
    amber
    [to]
    martini3001

    [from blocks]
    C-ter
    [to blocks]
    C-ter

    [from nodes]
    N
    HN

    [from edges]
    HN N
    N CA

    [mapping]
    CA BB
    C  BB
    O  BB
    OXT BB
