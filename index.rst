..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

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

.. sectnum::

..
  Based on the Atlassian Postmortem Template:

  https://www.atlassian.com/incident-management/postmortem/templates

Incident summary
================

High packet loss impacting TCP throughput was observed starting on 2020-05-21.
The LHN had previously been tested by Ampath, ESnet, and NCSA engineers using
the performance tools between a DTN at La Serena and a DTN at NCSA.  During
that testing period, engineers were able to determine the nuances of the tools
being used (iperf3) and the underlying hardware.  This allowed engineers to
properly tune the DTN servers, as well as TCP/UDP settings on the testing tools
to obtain the highest throughput possible.   During this initial testing we
determined that the QOS on the 2x20 Gbps links provided by ESnet caused varying
degrees of packet loss and throughput bandwidth of the tests.  During those
tests we were observing around 10Gbps of throughput in a theoretical 20Gbps
link.   After further testing, configurations were found that resolved this
issue and we were able to fully maximize the links fully.  These results were
documented as part of the network baseline.

Leadup
======

An ``iperf3`` run was performed from the host ``comcam-arctl01.ls.lsst.org`` on
Thursday, 2020-05-25. This hosts has an interface directly connected to
``lsst-br`` (LHN border router in the base datacenter).  This run reported the
expected ~150mbit/s throughput over TCP for traffic egress from LS to NCSA for
a Linux host with default tcp/ip/nic parameters over a ~190ms path.  The value
is expected to be grossly similar over commodity and the LHN as the latency is
roughly equivalent.  This testing was done as a final check that there were no
major problems with the hosts network configuration and in preparation to done
some basic TCP/IP parameter tuning the following day.

Fault
=====

On Friday, 2020-05-22, as part of preparing to tune the TCP/IP stack on
``comcam-arctl01.ls.lsst.org`` additional ``iperf`` runs were performed, and
the TCP throughput had dropped to < 2mbit/s.

.. code-block:: bash

  [jhoblitt@comcam-arctl01 ~]$ for i in 2 ; do sh -c "numactl -C $(($i-1))
  iperf3 -c starlight-dtn.ncsa.illinois.edu -t 60 -i 10 -b1000m -l8912-T
  s$i -p $((5000+$i)) -B 139.229.140.4"; done
  Connecting to host starlight-dtn.ncsa.illinois.edu, port 5002
  [  4] local 139.229.140.4 port 41340 connected to 141.142.141.66 port 5002
  [ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
  [  4]   0.00-10.00  sec  2.28 MBytes  1.91 Mbits/sec   12   35.0 KBytes
  [  4]  10.00-20.00  sec  1.83 MBytes  1.53 Mbits/sec    9   43.7 KBytes
  [  4]  20.00-30.00  sec  1.62 MBytes  1.36 Mbits/sec   11   52.4 KBytes
  [  4]  30.00-40.00  sec  2.35 MBytes  1.97 Mbits/sec    7   96.1 KBytes
  [  4]  40.00-50.00  sec  1.88 MBytes  1.58 Mbits/sec   12   43.7 KBytes
  [  4]  50.00-60.00  sec  2.25 MBytes  1.89 Mbits/sec   11   78.6 KBytes
  - - - - - - - - - - - - - - - - - - - - - - - - -
  [ ID] Interval           Transfer     Bandwidth       Retr
  [  4]   0.00-60.00  sec  12.2 MBytes  1.71 Mbits/sec   62             sender
  [  4]   0.00-60.00  sec  12.1 MBytes  1.69 Mbits/sec
  receiver

  [jhoblitt@comcam-arctl01 ~]$ for i in 2 ; do sh -c "numactl -C $(($i-1))
  iperf3 -u -c starlight-dtn.ncsa.illinois.edu -t 60 -i 10 -b1000m
  -l8912-T s$i -p $((5000+$i)) -B 139.229.140.4"; done
  Connecting to host starlight-dtn.ncsa.illinois.edu, port 5002
  [  4] local 139.229.140.4 port 34289 connected to 141.142.141.66 port 5002
  [ ID] Interval           Transfer     Bandwidth       Total Datagrams
  [  4]   0.00-10.00  sec  1.16 GBytes   993 Mbits/sec  139223
  [  4]  10.00-20.00  sec  1.16 GBytes  1.00 Gbits/sec  140262
  [  4]  20.00-30.00  sec  1.16 GBytes  1.00 Gbits/sec  140264
  [  4]  30.00-40.00  sec  1.16 GBytes  1.00 Gbits/sec  140262
  [  4]  40.00-50.00  sec  1.16 GBytes  1.00 Gbits/sec  140262
  [  4]  50.00-60.00  sec  1.16 GBytes  1000 Mbits/sec  140259
  - - - - - - - - - - - - - - - - - - - - - - - - -
  [ ID] Interval           Transfer     Bandwidth       Jitter
  Lost/Total Datagrams
  [  4]   0.00-60.00  sec  6.98 GBytes   999 Mbits/sec  0.016 ms 58791/840532 (7%)
  [  4] Sent 840532 datagrams
  iperf Done.

Impact
======

As there were no on going attempts to move large quantity of data primarily
TCP/IP tuning efforts were disrupted.  However, the upcoming Ops Rehearsal,
which at that time was planned to start June 1st, would have been disrupted.

Detection
=========

The problem was detected "by chance" as manual tests between the base
data-center and NCSA are not made on a regular basis.  Nor was there automated
end to end throughput tested as the planned perfSonar node at NCSA hasn't been
installed or added the an ampath maddash grid.

Response
========

Notification of the problem was made via the lsstcorp slack ``#rubinobs-lhn``
channel on Friday, 2020-06-22.  As this was the day prior to a US 3-day holiday
weekend, email to the ``lsst-net@lists.lsst.org`` mailing list was intentional
withheld until Tuesday, 2020-06-26.  Multiple members of the Chilean IT team
ran ``iperf`` tests periodically over the holiday weekend and did not observed
a significant change in behavior.  Shortly after email notification went out,
several members of the LHN team began coordinated debugging efforts via email
and slack.

Recovery
========

During the process of adding additional servers to the LHN, LSST IT admins were
performing speed tests using iperf3 between the BDC located in La Serena, to a
server located at NCSA in Urbana Illinois. During the initial testing, packet
loss, and low transfer rates were reported. After further investigation, a lack
of tuning on the servers seemed to be a direct cause of the packet loss. The
lack of tuning was causing packets/frames to be dropped inside of the server
and network interface before they were being sent out onto the LHN. After
proper tuning (increasing various buffer sizes, and adjusting the testing
software) this resolved most issues.

When the problem was first discovered, it was decided to use the existing DTN
that engineers had previously used to baseline the circuits. This was to go
back to the original baseline to see if the performance of the LHN had in fact
changed. Using those DTN, engineers were able to observe that there was in fact
a deterioration of performance when performing the tests.  After further
investigation it was determined that the DTN had been reconfigured by LSST IT
and many of the previous tuning parameters had been reset.  Once this was
corrected we were able to continue testing with the DTN.  While performance
improved engineers were still seeing lower than expected performance.

The final issue was that was uncovered revolved around the ESnet router.  An
ESnet router was recently replaced in Chicago.  During this replacement, the
tuning parameters that had previously been applied by ESnet engineers were not
applied to the new router.  This caused some variability in the testing
results, once all the DTN had been properly configured.  Once the configuration
was applied to the ESnet router, results returned back to the original baseline
as expected.

Timeline
========

2020-05-22 - degraded TCP performance observed; ``#rubinos-lhn`` notified

2020-05-23 - degraded TCP performance re-observed

2020-05-24 - degraded TCP performance re-observed

2020-05-25 - degraded TCP performance re-observed

2020-05-26 - degraded TCP performance re-observed; email notification sent

2020-05-26 - ampath <-> starlight path reverted to vlan311

2020-05-27 - switched to vlan312 for testing; percussive maintenance on routers; comcam server tuning

2020-05-28 - additional network testing

Root cause
==========

* Possible Packet loss on the backup circuit (vlan 312).  Unknown cause and can not replicate.
* Properly tuning missing on comcam server.
* Missing configuration on ESnet server caused some bandwidth restrictions around 10Gbps.


Recurrence
==========

No prior example of a similar fault is known.

Lessons learned
===============

* [lack of] IP stack tuning can cause high apparent ``iperf`` packet loss during an UDP flood that is actually occurring largely on the tx host (and possibly also on the rx host).
* all team members need access to ``iperf`` test hosts at each significant hop along the network.
* automated perfSonar testing between all hops and end-to-end is needed for timely detection and would simplify debugging
* automated monitoring of the path may be helpful
* a formal postmortem process needs to be identified

Recommendations
===============

Reproducible tests are required for network validation and monitoring.  The
DTNs should be well documented with any tuning parameters that are applied to
the servers and a change log should be kept when any changes do occur to the
system.  Also well documented testing procedures should exists for that exact
tests that are to be run (scripts, with documented CLI parameters should kept).
This allows engineers who are not familiar with previous testing, to run the
same scripts and allow for a more rigorous testing procedure.  Continuous
performance testing using PerfSonar should also be implemented to create a
baseline of the network and constant testing against that base line should be
performed.  This also allows engineers to test from various points in the
network to help isolate the problem.  This is planned, but at this time not
implemented fully yet.
