% Planning for the second phase of Track data product revision
% Giuseppe Cerati (cerati@fnal.gov)
  Gianluca Petrillo (petrillo@fnal.gov, editor)
  Erica Snider (erica@fnal.gov)
% March 28, 2017


Scope
======

The goal of this discussion is to resume the planning for the continuation of
the tracking revolution in LArSoft.

The plan to redesign tracking in LArSoft as described at
[LArSoft coordination meeting][LArCoords20170117] includes three stages:

1. definition of the new data products, minimal impact on the interface
2. basic tools to write and navigate the new objects, clean up of the data
   product interface, code update to create the new objects
3. extension on the navigation tools

That plan as written in the slides was approved by the convened at the meeting.
The first phase was completed at the end of January 2017.

The goals of next phase ("phase II") need to be defined in detail.
This must include:

* the tools in line for delivery
* the interfaces being changed
* the transition strategy and plan to adopt them



Target
=======

The hierarchy of classes that emerged from "phase I" has two main data products:
`recob::TrackTrajectory`, meant to be the result of 3D pattern recognition
algorithms, and `recob::Track`, meant to be the result of fitting of the former.


The roles of `recob::Track` and `recob::TrackTrajectory`
---------------------------------------------------------

The consensus among the convened is to shift the paradigm toward the exclusive
production of `recob::Track`. While pattern recognition algorithms should still
create `recob::TrackTrajectory` objects, these are expected to be fitted and the
corresponding `recob::Track` saved.
The main advantage of this enforcement is the association of data products,
which is going to happen only via `recob::Track`. For example, if a
`recob::PFParticle` is going to have track-like objects associated with it,
those are expected to be `recob::Track`.
This simplifies the set of interfaces we directly support and maintain, and
provide downstream algorithm writers a smaller set of cases to be supported.
This is not meant to prevent people from using `recob::TrackTrajectory`.

It is important to stress that a `recob::Track` object can describe the result
of a failed attempt to fit a trajectory. That is to say that, when fitting a
collection of trajectories, we do expect one track per trajectory to be present,
potentially with the information that the fit failed coded into the track
object.



Deliverables
=============

In this section, we list the deliverables of this project.
Here they are not bound to any phase of the project: such assignment is
described in the following section.


Tools
------

Phase II includes tools for creation and for access of the new data products and
connections. These are two distinct and fairly independent groups.

Creation tools are aimed to facilitate the creation of the new data products in
a consistent way fulfilling the data product requirements, and they are aimed to
be used by existing and future algorithms.

Access tools facilitate the navigation of the related objects across data
products, and provide an uniform interface.
One specific goal of the toolset is to allow for an interface able to seamlessly
access data from either `recob::Track` or `recob::TrackTrajectory`, without the
user even being aware of the source of that information.

Creation tools may be designed from simpler, lower level to more comprehensive,
higher level:

1. check of input for the creation of a track or trajectory: using as input
   separate collections of positions, momenta _(optional)_, flags _(optional)_,
   and `recob::Hit` _art_ pointers, a function returns if these fulfill the
   requirements for a track or trajectory (e.g. coherent number of entries,
   minimum number of points...)
2. creation of a full object: starting from the same input as before, a
   fully featured `recob::TrackTrajectory` or `recob::Track` (which requires
   additional input) is returned
3. creation of the full object and associations: starting from the same input as
   before, plus other objects _(optional)_ as vertices or reconstructed space
   points or seeds, data product collections are updated to include the new data
   objects and their associations
4. same goal as in the previous point, with the track or trajectory object being
   progressively put together point by point, with as input a single position,
   momentum _(optional)_, flags _(optional)_ and an hit pointer for each new
   point

Access tools can also be tiered with increasing complexity and functionality:

1. interface exposing the `recob::TrackTrajectory`, extracting its information
   from either a `recob::TrackTrajectory` or a `recob::Track`; depending on our
   decision, user code may be required to adopt a templated approach, or not
2. interface exposing the above, and `recob::Track` interface; a protocol needs
   to be established to define the discovery of available information
3. interface as above, with additional point navigation facilities (e.g. skip
   to the next point which is on the fit)
4. interface exposing the information above, and offering the associated hits
   and residuals
5. as above, plus access to other type of associated objects (vertices,
  clusters, etc.)

Both the categories will need some connection to the entire collection in order
to access the correct associated objects. The way this is achieved is an
implementation detail to be designed.


Code updates
-------------

Code updates will be sometimes mandatory, sometimes recommended, including:

* adoption of writing tools by existing `recob::Track` producers
* adoption of access tools in algorithms working with `recob::Track`

The modules producing `recob::Track` have now a choice: they can produce a
complete `recob::Track`, or just a `recob::TrackTrajectory`. The former includes
fit-like quantities like $\chi^{2}$ and covariance at the two ends. Therefore
the choice should be driven by whether the algorithm is able to meaningfully
compute such quantities ("fit") or not ("pattern recognition"). We expect most
of the algorithms to fall in the latter category. Thanks to the access tools,
algorithms should face no penalty in usability by choosing to produce
`recob::TrackTrajectory` objects.

For this to actually happen, the algorithms should be updated to utilise an
access tool rather than `recob::Track` directly. Even when the algorithm
explicitly requires `recob::Track` information, the tools may have a value in
facilitating the navigation of the track with the associated objects (hits,
vertices, etc.).



Delivery plan
==============

The original plan includes within phase II the delivery of some lower level
tools and the complete update of the code.

For the creation tools, a baseline may be identified in the delivery of tools to
cover the points (1) and (2) documented above, with (3) a desireable addition.

For the access tools, a similar baseline can be defined in points (1) and (2).
This is expected to suffice to upgrade the existing code.

The code to be updated may be in principle divided in LArSoft code and
experiment code. In practice, it is easy to imaging dependencies between the two
that in specific cases make it impossible for the latter to happen without the
former, and/or make it necessary for the latter to immediately follow the
former.

Therefore, while the division is still useful, it may be necessary to soften the
transition on LArSoft code, so that when possible it temporary keeps a legacy
interface too. This is sometimes not possible. The cases we have in mind,
where only the return value of a interface function changes, are expected to be
easy to roughly address in the depending code. In this example, the Authors
might choose to convert their experiment code to convert the returned data type
into the old one. The recommended approach is to upgrade the code to natively
use the new data types, which can be between trivial, tedious and plain hard.
We will accumulate experience and produce documentation on the process while we
do the very same operation on LArSoft code.

We consider the upgrade of the code using the data objects a higher priority
than the one creating them, to avoid a scenario where the old algorithms
produce a `recob::TrackTrajectory` which the downstream code is not yet ready to
use.

The complete upgrade of this downstream code may be a vast effort that we have
not quantified. It is advisable to have it tiered as well. The recommended
baseline is that no LArSoft code should be left depending on the legacy
interface.
The amount of that being achieved via adoption of the new tools versus the
simple update of direct interface to the data objects is defined above that
baseline, except that at least one use case needs to be fully updated to prove
the new interface suitable and to earn the experience that translates into
upgrade instructions.

A small part of the code using the old interface will have to be immediately
upgraded as the old interface undergoes changes mandatory for technical reasons.
Unless it proves to be a prohibitive effort, we are going to perform this
upgrade for LArSoft code and for the experiment code as well.
Beyond that, the goal is for LArSoft code not to depend on the legacy interface
at all, as implied by the baseline defined in the previous paragraph.
After that is achieved, the legacy interface should disappear.
We would follow the custom of deprecating the old interface and then remove it
after the expiration of the transition time. At that time, we will also remove
the interface that is already deprecated.


The proposal is summarised in the following:

1. delivery of access tools, as described in this section
2.  
    1. delivery of creation tools, as also described in this section
    2. update of LArSoft code using `recob::Track`, collecting the procedures
       into upgrade instructions for users and experiments
3.  
    1. update of LArSoft code currently creating `recob::Track`, collecting the
       procedures into upgrade instructions for users and experiments
    2. disruption of the old interface
4. removal of the legacy interface after a transition period

The sub-points may be executed independently from each other, while the main
points need to be sequentially performed.



Related material
=================

* [task #14281][Issue14281]
* presentation ["New tracking data products"][LArCoords20170117] at LArSoft
  Coordination Meeting on January 17, 2017



History of this document
=========================

- Version 1.0,
    from a meeting on March 21, 2017, with Giuseppe Cerati and Gianluca Petrillo
- Version 1.1 (this document),
    from a meeting on March 28, 2017, with Giuseppe Cerati, Gianluca Petrillo
    and Erica Snider



About this document
====================

This document is written in pandoc markdown format. It can be rendered in PDF
and HTML format by running respectively:
    
    pandoc -o 20170321.pdf 20170321.md
    pandoc -o 20170321.html 20170321.md
    



[Issue14281]:        https://cdcvs.fnal.gov/redmine/issues/14281
[LArCoords20170117]: https://indico.fnal.gov/getFile.py/access?contribId=1&resId=0&materialId=slides&confId=13672
