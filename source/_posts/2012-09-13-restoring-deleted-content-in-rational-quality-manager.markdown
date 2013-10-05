---
layout: post
title: "Restoring deleted content in Rational Quality Manager"
date: 2012-09-13 15:44
comments: true
categories: 
---

Recently I had an issue where a tester had deleted a number of artifacts from RQM by mistake and wanted to know how to restore them. My first port of call was the [jazz.net][1] forums hoping to find that RQM has an un-delete button that I had somehow missed, however I quickly found that this is [not the case][2].

Thankfully the developer that responded in the thread was also kind enough to provide the necessary information to allow you to manually restore records through the REST API:

{% codeblock %}
https://server:port/qmservice/com.ibm.rqm.planning.common.service.rest.ITestCaseRestService/archiveTestCases?uuids_versionable=<test_case_uuid>&processArea=<project_uuid>&archive=false
{% endcodeblock %}


This is certainly a step in the right direction, however it leads to the next obvious question of how do I find the UUIDs of my test cases and project area? My first line of investigation was the [OSLC API][3] however this quickly hit a dead-end when I found that the OSLC API does not return deleted records.

## Reportable REST API to the rescue

In the past I've had occasion to download the contents of individual test cases in XML format using the [RQM URL Utility][4] tool. This tool is invaluable when you are trying to write a custom configuration file for the [RQM Word and Excel][5] importer because it enables you to see the identifiers for custom sections. The examples you see in the URL utility documentation are all based on the [RQM Reportable REST API][6], which provides a means of both listing and viewing individual records within RQM.

Looking through the documentation I was able to find the solution to seeing the deleted records: the [includeArchived][7] query parameter. Performing a GET on the test case feed URI with this parameter set to true I was able to get an XML response that contained all of the test cases in my RQM project area, both active and deleted. The last remaining issue was to get the UUID of each of the test records, which is also catered for in the API via the [metadata][8] query parameter.

All of this leads you to a query shape of:

{% codeblock %}
https://server:port/qm/service/com.ibm.rqm.integration.service.IIntegrationService/resources/<project-alias>/<resource-type>
{% endcodeblock %}

Performing a GET on the above URL leads to a result-set that contains entries similar to the following:

{% codeblock lang:xml %}
<entry xmlns="http://www.w3.org/2005/Atom">
 <id>_KSwzJeLrEeGnZrjoVKQNBw</id>
 <title type="text">My Test Case</title>
 <summary type="text"/>
 <updated>2012-08-13T13:52:08.259Z</updated>
 ...
</entry>
{% endcodeblock %}

The UUID of the test case is the value of the `id` element.

The last remaining step is to find the UUID of the project the records are part of.  I've found that the easiest way to obtain this is to simply open the "Manage This Project Area" page within RQM, then get the UUID from the `itemId` query parameter.

## Putting it all together

Now that we have both the UUID of the test case and the project area we can replace the values in the URL I mentioned at the start of the article and perform an HTTP POST to restore the test cases on the server. To perform a POST you can either use a command line utility such as [cURL][9] or a tool such as the Firefox [Poster][10] addon. Below is an example of the cURL command to use:

{% codeblock %}
curl -X POST https://rqmserver:9443/qm/service/com.ibm.rqm.planning.common.service.rest.ITestCaseRestService/archiveTestCases?uuids_versionable=_KSwzJeLrEeGnZrjoVKQNBw&processArea=_W1iVMBuDEdy4euo1O8SrOg&archive=false
{% endcodeblock %}

Overall this is quite a complicated process for an operation that should really only take a couple of clicks through the user interface. Hopefully this article will help someone avoid either having to manually re-type a bunch of information or worse, restoring their database from a backup. ~~With any luck IBM will plan this functionality in for a release sometime soon, however after being open for [15 months][11] at the time of writing this, I feel it's already long overdue.~~

**Update 29/08/13:** As of version 4.0.3 RQM now has the capability to restore deleted content (as well as permanently deleting records). For users of prior versions you will still need to use the steps I've documented above.


[1]: https://jazz.net/forum
[2]: https://jazz.net/forum/questions/74023/how-to-reinstate-a-deleted-test-case
[3]: http://open-services.net/
[4]: https://jazz.net/wiki/bin/view/Main/RQMURLUtility
[5]: https://jazz.net/wiki/bin/view/Main/RQMExcelWordImporter
[6]: https://jazz.net/wiki/bin/view/Main/RqmApi
[7]: https://jazz.net/wiki/bin/view/Main/RqmApi#includeArchived
[8]: https://jazz.net/wiki/bin/view/Main/RqmApi#metadata
[9]: http://curl.haxx.se/
[10]: https://addons.mozilla.org/en-us/firefox/addon/poster/
[11]: https://jazz.net/jazz02/resource/itemName/com.ibm.team.workitem.WorkItem/52743