.. note::

   **This technote is not yet published.**


Abstract
========

Presents a sketch of a core RSP service that uses a combination of information from Kubernetes and from individual service to satisfy the "/availability" endpoints of all our IVOA-style services associated with a single RSP instance.
Also presents a proposal for a core page on the RSP web presence that displays service status based on this information, and, for the API Aspect, links to additional per-service information.

Motivation
==========

Three strands come together in this proposal:
most users of contemporary web-based service environments expect something like a status page
for the set of services run by a provider;
all contemporary IVOA standards either require as a mandatory element the provision of
an ``/availability`` endpoint for each service, returning a standardized response; and
because we run all the RSP services in a uniformly-managed Kubernetes environment,
basic information about service availability is available centrally.

Detailed Background
===================

IVOA Considerations
-------------------

The *IVOA Support Interfaces* standard, commonly known as "VOSI" [IVOA-VOSI2017]_,
specifies a set of common interfaces that all IVOA-compliant services shall,
or should (some are optional) provide.
Among these are a service-availability reporting interface, with a specified REST
binding and XML response.
This is a mandatory component of the VOSI standard.
The intent of this is to make it possible for service status to be reported uniformly,
facilitating the writing of client code that works across a range of services.

.. [IVOA-VOSI2017] Matthew Graham et al., 2017. IVOA Support Interfaces, Version 1.1.
                   IVOA Recommendation 2017-05-24, Grid and Web Services Working Group.
                   `<https://ivoa.net/documents/VOSI/20170524/REC-VOSI-1.1.html>`__

Availability XML response
^^^^^^^^^^^^^^^^^^^^^^^^^

The specific content of the response from the ``/availability`` endpoint of a VOSI-based service is described in the standard as follows:

    This interface indicates whether the service is operable and the reliability of the service for extended and scheduled requests.
    The availability shall be represented as an XML document in which the root element is http://www.ivoa.net/xml/Availability/v1.0#availability.
    This element shall contain child elements providing the following information:

    - ``available`` - whether the service is currently accepting requests
    - ``upSince`` - duration for which the service has been continuously available
    - ``downAt`` - the instant at which the service is next scheduled to be unavailable
    - ``backAt`` - the instant at which the service is scheduled to become available again after down time;
    - ``note`` - textual note, e.g. explaining the reason for unavailability.

    The elements ``upSince``, downAt``, ``backAt`` and ``note`` are optional.
    The ``available`` element is mandatory.
    There may be more than one ``note`` element.

    The XML document shall conform to the schema given in appendix B of this specification.

    When reporting availability, the service should do a good check on its underlying parts to see if it is still operational and not just make a simple return from a web server, e.g., if it relies on a database it should check that the database is still up.
    If any of these checks fail, the service should set available to false in the availability output.

    If a service is to be online but unavailable for work (e.g., when a service with a work queue intends to shut down after draining the queue) then the service should set available to false.

    There are no special elements in the availability document for the contact details of the service operator. These details may be given as a ``note`` element if they are known to the service.

    In the REST binding, the availability shall be a single web resource with a registered URL.

    All VO services shall provide this interface.

The full `XML schema <https://ivoa.net/documents/VOSI/20170524/REC-VOSI-1.1.html#tth_sEcB>`__ is
available in the standard.

Given a list of services and their availability REST endpoints, a client can easily assemble
a table or "Christmas tree" of status values for the services.


The problem of self-reporting
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Superficially implied in some of the discussions of the VOSI endpoints in higher-level
IVOA standards such as TAP is that the ``/availability`` endpoint **for** a service is an
endpoint **of** the service, i.e., provided by the same system component that provides
the substantive service.

The VOSI statement that when "reporting availability, the service should do a good check 
on its underlying parts to see if it is still operational and not just make a simple
return from a web server" emphasizes the role of the service itself in this respect.
However, a service can be down for reasons other than the unavailability of a lower-level
service on which it depends, and these cases bring the contradictory suggestion that a
service somehow successfully respond to an ``/availability`` query even when it is
otherwise down.

In this note, we propose a two-level mechanism: a primary level in which a robust
central supervisor maintains a basic "aliveness" status for all services, and an optional
second level in which a superficially "alive" service can be consulted for a more
substantive assessment of its own health.


Kubernetes Deployment Environment
---------------------------------

In the highly automated and uniform Kubernetes deployment environment being developed
for the RSP, all services are represented by Kubernetes pods, with the Kubernetes
controller able to report on the "running" status of any pod.
In addition, in the Kubernetes environment, all external contact with the running
services passes through "ingress rules" which connect externally visible endpoints
to specific points in the back end.

A planned future elaboration of the system envisions the provision of an authenticated
"throttling" layer which is capable of rate-limiting requests to specific services,
potentially conditioned on the identity of the requesting user, and replying with
an HTTP 429 status.

The Kubernetes controller is likely to be able to report information relevant to
the reporting of the ``upSince` time stamps in the VOSI specification.

Planned downtime messaging
^^^^^^^^^^^^^^^^^^^^^^^^^^

As part of a 2021 revision of the RSP front page, the SQuaRE group is working on a
mechanism that would allow posting urgent messages to users, including human-readable
service status messages.
This mechanism is likely to be able to hold information that would be relevant to
the reporting of the ``downAt`` and ``backAt`` time stamps in the VOSI specification,
for planned outages.


Proposal
========

We propose the creation of a central availability service, deployed in Kubernetes
for each RSP instance, and responsible only for responding to VOSI-Availability
REST queries for all the relevant RSP services on that instance.
This should of course include every formally IVOA-compliant service, such as TAP
and SODA, but could usefully be extended to all public-facing RSP services, even
ones for which VOSI is otherwise irrelevant, e.g., JupyterLab and Firefly.

This service would have a REST interface along the lines of
``https://(address)/api/central-availability/(service-name)`` .

This service would return the prescribed XML response for every RSP service,
by default based only on information known to the Kubernetes controller and on
a future repository of planned-downtime information.

Optionally, for specifically configured services, the central availability service
would, when it believed the underlying service to be up, call through to that
service's own, internal ``/availability`` endpoint for a more thorough assessment,
following the recommendation in the VOSI standard.
This would allow, for instance, the TAP service to check that the underlying
Qserv database was responding to simple requests, or the SODA service to check
that the production Butler is functional.

Kubernetes ingress rules would be used to redirect IVOA-style ``/availability``
endpoints "below" the specific service address,
such as https://(RSP-external-address)/api/tap/availability,
away from the service-specific availability endpoint, and
to the corresponding endpoint on the central availability service.

The central availability service could be directly exposed externally,
or only through the redirected external per-service endpoints; this is TBD.
An advantage to exposing it directly is that it provides a natural home for
delivering status information for non-IVOA services (e.g., Nublado) that are
not constrained to have their endpoints registered in VOSI-like ways.
Any "internal" service-specific availability endpoints, provided by the specific
IVOA service pods themselves, would not be exposed externally at all.

For services for which throttling / rate-limiting are enabled, if that is
implemented in a generic way across the RSP's web services, the central
availability service should be able to be extended to return a message
concerning whether rate-limiting is currently being applied.

Finally, with a uniform availability service in place, it is easy to imagine
writing a lightweight "dashboard" / "Christmas tree" status display for all
the components of the RSP.

The proposal includes the following specific elements:

Basic /availability XML retrieval from Kubernetes controller
------------------------------------------------------------

The core element of the proposal is a service which can be configured with
a list of services to represent, and the logic necessary to query the
Kubernetes controller for the present state of a service and translate that
into the ``available`` and ``upSince`` attributes of the VOSI-Availability
data model.

If a centralized repository of information about planned outages is developed
in order to support messaging through the new RSP home page and framing,
this could also be queried in order to support delivery of information about
planned outages via the ``downAt`` and ``backAt`` attributes for scheduled
downtimes.

An instance of the ``note`` attribute should be generated representing the
basic assessment that the service appears to be running.
If a planned outage is known to the system, a human-readable message about
the outage, e.g., "planned maintenance 2024-04-01 14:00Z through 2024-04-02 16:00Z",
or "down for rollout of DR3 2026-10-09" could be included in the ``note``.

The service would need to be able to format this information in the VOSI-prescribed
(very simple) XML format.

Since the author is aware of the existence of anti-XML attitudes in the RSP
developer community, we note that the central availability service could be
designed to respond both in JSON and XML.
One could imagine 
``https://(address)/api/central-availability/(service-name)``
responding in JSON and
``https://(address)/api/central-availability/(service-name)/xml``
in XML, with the per-service availability endpoints redirected to the latter.

This level of functionality alone, without any of the following components,
would already be useful.

Layered call to service-specific availability endpoint
------------------------------------------------------

In order to support the VOSI recommendation that the availability response reflect
a real check by the service of whether it can function, a second phase of 
development of the central availability service should add a per-service option
to call through to the specific service's own availability endpoint, and to
integrate its response with the central service's notion of whether the specific
service is up.

If the Kubernetes controller shows that the underlying specific service appears
to be running, the call-through would be made.
Failure to respond within a designated timeout, or a response code other than 200,
should lead to reporting the service as unavailable after all.
If the service returns a report stating that it is not available, that should 
supersede the central service's notion of whether it is running.
Any of these cases where the actual service availability appears to conflict with
the central Kubernetes status should also be recorded as an error to be flagged
to the RSP operations team, of course.

Layered query options
^^^^^^^^^^^^^^^^^^^^^

**Mandatory, VOSI-XML response**

The call-through query must at a minimum support receiving a VOSI-standard XML
availability message.
This is because we are using existing server implementations for some of our
services, and they already support this interface.
Any ``note`` elements returned should be passed through to the external caller,
as the schema supports multiple ``note``s.
If the service returns ``upSince``, ``downAt``, or ``backAt``, TBD logic would
be required to harmonize these with the central service's ideas of these.
(This may be unlikely to arise in practice; these elements are not returned by
most open-source IVOA server implementations.)

**Alternate query option**

As a configurable option, the call-through may also be designed to recognize
an alternate, much simpler, query response.
The motivation is to support the implementation of lightweight ancillary
services such as DataLink-followers, which do not need to be fully VOSI-compliant
in their own right.
An example of a stripped-down response model for the call-through might be:

- "yes, available" is represented by a 200 status and an optional plain-text
  message body, which, if present, is forwarded as an additional ``note``
  element in the response from the central availability service, and
- "no, not available" is represented by a failure code (503 might be appropriate)
  and, again, any plain text in the response body is forwarded as a ``note``.

The choice between the query options could be configured on the central
availability service, or it could be implemented as a run-time recognition of
a plain-text or null 200 response as acceptable.

Caching
^^^^^^^

In order to avoid excessive load on the back-end service, the central availability
service could cache the underlying service's response for an appropriate period,
e.g., 60 seconds, and limit the call-through to when the cache has expired.

It is TBD whether the central availability service would *continually* (upon cache
expiration) ping the underlying services so configured (providing internally useful
status information and enabling internal alerts if the central service thinks a
service is up but it reports itself as non-functional), or only in response to an
external request.

Ingress configuration and possible directory service
----------------------------------------------------

It is proposed that ``https://(RSP-external-address)/api/X/availability``, the normal
endpoint for a VOSI service ``X``, be redirected via ingress rule to 
``https://(availability-service-address)/api/central-availability/X``.

This proposal expresses a preference for, but does not insist on, making the
central availability service directly available externally at a single endpoint,
exposing the service-specific endpoints below that externally as well --
in addition to satisfying the required per-service VOSI-availability endpoints
by redirecting them to the appropriate URL.
That is, ``(availability-service-address)`` might be the same as ``(RSP-external-address)``.

If so, then the possibility appears of allowing the base URL
``https://(availability-service-address)/api/central-availability``
to serve as a REST-standard directory of services, returning a list of the
valid endpoints one level down.
This proposal does not take a position on that.

A&A considerations
------------------

The VOSI standard states "the availability binding[s] must be available to
anonymous requests".

In the "naive" implementation model where each service provides its own externally
accessible availability endpoint, this places each service in the position of 
handling external data from unauthenticated clients directly.
This increases the attack surface of the RSP and might require more careful
service-by-service vetting of their response to availability queries.

In the proposed model, the unauthenticated ``/availability`` requests are all
handled by the central service.
No individual service would be responsible for processing any unauthenticated
availability request URLs.
For an unauthenticated external request, the central service's call-through to
the specific service, if implemented, might use a non-privileged internal
service identity to authenticate the call-through request.
For an authenticated external request, that identity could simply be passed
through.
This design allows for possible future implementations where the call-through
response is customized to a specific user.

Dashboard
---------

This proposal envisions that the central availability service would be used to
construct a status dashboard for all the RSP services.
This could, but need not, utilize the XML responses; it is conceivable that a
Rubin-private response format could be used to populate the dashboard.

Note, however, that any external client using IVOA standards could also 
construct a basic dashboard for the RSP, which seems like a positive feature.

Since such a dashboard would *per se* incorporate a list of all available API
services, it might also be a good jumping-off point for additional information
on each service.
This might include both a user manual and an OpenAPI/Swagger-style "try out
this service" page.
(The author would very much like to see such a capability in the long run.)


..
  Technote content.

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. Do not include the document title (it's automatically added from metadata.yaml).

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
