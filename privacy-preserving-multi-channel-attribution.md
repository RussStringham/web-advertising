#Privacy Preserving Multi-Channel Attribution

## Motivation
This proposal modifies [Cross-Browser Anonymous Conversion Reporting](https://github.com/w3c/web-advertising/blob/master/cross-browser-anonymous-conversion-reporting.md) by adding support for other marketing channels beyond ads that may have contributed to a conversion. In addition to the web ads that a user viewed and/or clicked from their various browsers, the advertiser may be aware of other interactions that may have contributed to a conversion. For example, the user may have received emails promoting the site and/or specific products, or push notifications to their mobile device. They may have contacted a call center or interacted with the site via a chat session. Attribution should incorporate interactions from all of these channels.

### Disclaimer
As claimed by the Cross-Browser proposal, this is not a perfect proposal. This is an early stage idea. The intention is to start a conversation with the broader community, and collaboratively work to improve on this starting point, to find even better solutions to this problem.

### Author:
* Russell Stringham (rstringh@adobe.com)

## Proposal Overview
When a conversion occurs, the advertiser will call the appropriate conversion measurement API in the browser. There are a number of proposed conversion measurement APIs that could be used including [Private Click Measurement](https://wicg.github.io/ad-click-attribution/index.html) (Webkit/Apple), [Conversion Measurement API](https://github.com/csharrison/conversion-measurement-api) (Chrome/Google) and [Private Lift Measurement](https://github.com/w3c/web-advertising/blob/master/private-lift-measurement-conceptual-overview.md). I will leverage the Chrome proposal for my example here, enhancing its [Conversion Measurement with Aggregation](https://github.com/WICG/conversion-measurement-api/blob/master/AGGREGATE.md#conversion-measurement-with-aggregation-explainer) idea to support attribution on multiple marketing channels.

When notified of the conversion, the browser will generate a conversion ID (a globally unique ID (GUID)) that it will return to the advertiser, along with its timestamp of the conversion time. It will also generate an attribution report that it will deliver at some point in the future to the advertiser as described in the Google paper. In Google's diagram (shown below) the Advertiser or its DSP would be the Reporting Origin. The attribution report will contain two encrypted files, one for each helper, with each file encrypted using the public key of its helper.

![Google Aggregation Service Diagram](https://github.com/WICG/conversion-measurement-api/blob/master/static/mpc-arch.png)

The encrypted attribution report files generated by the browser for each helper will contain a second GUID generated by the browser exclusively for this conversion report. Each file will also contain a data element for every ad from that advertiser that was viewed or clicked within the conversion window. The data elements should ordered by the time the ad views/clicks occurred, from oldest to most recent. In the first file, each data element will contain the channel type (ad view or click), the ad ID and the campaign ID for the ad. In the second file. each element will contain channel type and a timestamp indicating when this occurred. The second file will also contain the conversion ID, the conversion value reported by the advertiser and the time of the conversion.  Here is a sample version of these files for a single conversion: 

>**First File**

    FileGUID: "9875",
    Interactions: {
        { Channel: "view",  AdID: "abc", CampaignID: "xyz" },
        { Channel: "view",  AdID: "def", CampaignID: "pqr" },
        { Channel: "view",  AdID: "abc", CampaignID: "xyz" },
        { Channel: "click", AdID: "mno", CampaignID: "stu" }
    }
>**Second File**

    FileGUID: "9875",
    Interactions: {
        { Channel: "view",  Time: "2020/03/02-13:39:50" },
        { Channel: "view",  Time: "2020/03/03-09:22:37" },
        { Channel: "view",  Time: "2020/03/03-09:43:05" },
        { Channel: "click", Time: "2020/03/04-17:11:56" }
    },
    ConversionId: "1234",
    Conversion-value: "1000"
    Conversion-time: "2020/03/05-08:37:03"
    
### Ad Views/Clicks from other Browsers
Cross-Browser Anonymous Conversion Reporting describes a method of attributing ads seen in all browsers used by an individual to a conversion that occurs in one of the browsers. It requires that all associated browsers share all ads viewed in each browser with all the other browsers, so each knows the full add history of all of them. This proposal does not require that sharing of ad history, reducing the risk from a bad association of a browser with a user. It still relies on the technique used by that proposal, described in [Enabling browsers that belong to the same person to discover one another](https://github.com/w3c/web-advertising/blob/master/enabling-browsers-that-belong-to-the-same-person-to-discover-one-another.md#enabling-browsers-that-belong-to-the-same-person-to-discover-one-another), to do what the title says. However, once the association is made, the technique described for sharing ads is instead used for sharing conversions. When a conversion occurs in one of the associated browsers, it shares the advertiser, conversion ID and the conversion timestamp with the other browsers using the techniques outlined in that proposal.

When the other browsers are notified of a conversion, they check their ad history to see if the user viewed or clicked any ads from that advertiser during the conversion period (such as 30 days prior to the conversion up to the conversion time). If the browser finds any ads, it also sends a conversion report to the Reporting Origin (advertiser or their DSP). This report is identical in format to the report from the original browser, except that it doesn't include a conversion value or conversion time in the second file.

>**First File**

    FileGUID: "7890",
    Interactions: {
        { Channel: "view",  AdID: "jkl", CampaignID: "pqr" }
    }
>**Second File**

    FileGUID: "7890",
    Interactions: {
        { Channel: "view",  Time: "2020/03/03-22:38:13" }
    },
    Conversion-id: "1234"
    
This attribution report should include a rough timestamp indicating when the conversion occurred (ideally the hour it occurred, but this proposal would work if it only specifies the day) that is readable by the Reporting Origin. This will help the Reporting Origin to group it with the original conversion report that may have arrived a few days earlier, in the case where the user doesn't use this other browser for a while after the conversion, even though the reporting origin cannot set the conversion ID.

### Other Channels
When the conversion occurs, the advertiser can also create other conversion reports for their other channels. Technically, these don't need to be encrypted, since the advertiser already has access to this information. If the advertiser contracts with a service provider for their email functionality, that service provider might generate a report containing email interactions with the user who converted, such as the following:

>**First File for Emails**

    FileGUID: "7788",
    Interactions: {
        { Channel: "email send",  EmailID: "123", CampaignID: "pqr" },
        { Channel: "email open",  EmailID: "123", CampaignID: "pqr" },
        { Channel: "email open",  EmailID: "123", CampaignID: "pqr" }
    }
>**Second File for Emails**

    FileGUID: "7788",
    Interactions: {
        { Channel: "email send",  Time: "2020/02/29-03:21:44" },
        { Channel: "email open",  Time: "2020/03/01-10:05:59" },
        { Channel: "email open",  Time: "2020/03/03-15:48:26" }
    },
    ConversionId: "1234"
    
The advertiser might also have internal data that they want to include in the attribution:

>**First File from Internal Advertiser Data**

    FileGUID: "5678",
    Interactions: {
        { Channel: "organic search",  SearchTerms: "widget1" },
        { Channel: "internal search", SearchTerms: "doohickey3" }
    }
    
>**Second File from Internal Advertiser Data**

    FileGUID: "5678",
    Interactions: {
        { Channel: "organic search",  Time: "2020/03/05-08:31:48" },
        { Channel: "internal search", Time: "2020/03/05-08:33:12" }
    },
    ConversionId: "1234"
    
### Helper Processing
If the conversion ID is an even number, the first file will be encrypted with the public key for helper 1 and the second file with the public key for helper 2. If the conversion ID is an odd number, it will switch the keys used for these files. As the key in this example is even, the first file of each set will be handled by helper 1, while the second will be handled by helper 2. When the advertiser sends the files to the helpers (along with the files for all the other conversions that have occurred during that time period so that results can be aggregated), it will specify an attribution algorithm to use.

For each unique conversion ID that it has, helper 2 will place the interactions that contributed to that conversion in order. Each interaction will also include the GUID of the file it came from and its position within that file. For our example conversion that will result in:

>**Helper 2 Processing: Path to Conversion**

    Interactions: {
        { Element: "7788-1", Channel: "email send",      Time: "2020/02/29-03:21:44" },
        { Element: "7788-2", Channel: "email open",      Time: "2020/03/01-10:05:59" },
        { Element: "9875-1", Channel: "view",            Time: "2020/03/02-13:39:50" },
        { Element: "9875-2", Channel: "view",            Time: "2020/03/03-09:22:37" },
        { Element: "9875-3", Channel: "view",            Time: "2020/03/03-09:43:05" },
        { Element: "7788-3", Channel: "email open",      Time: "2020/03/03-15:48:26" },
        { Element: "7890-1", Channel: "view",            Time: "2020/03/03-22:38:13" },
        { Element: "9875-4", Channel: "click",           Time: "2020/03/04-17:11:56" },
        { Element: "5678-1", Channel: "organic search",  Time: "2020/03/05-08:31:48" },
        { Element: "5678-2", Channel: "internal search", Time: "2020/03/05-08:33:12" }
    },
    ConversionId: "1234",
    Conversion-value: "1000"
    Conversion-time: "2020/03/05-08:37:03"

If the attribution algorithm is equal-credit (also known as linear), then all 10 interactions would get 10% of the conversion value of 1000 or 100 each. If the algorithm is a simple time decay, with a 10% reduction each calendar day, then the attribution model would calculate:

>**Helper 2: Computed Attribution Values for Simple Time Decay**
    AttributionComputation: {
        { Element: "7788-1", Channel: "email send",      Time: "2020/02/29-03:21:44", Rate:  "50%", Credit:  "63" },
        { Element: "7788-2", Channel: "email open",      Time: "2020/03/01-10:05:59", Rate:  "60%", Credit:  "76" },
        { Element: "9875-1", Channel: "view",            Time: "2020/03/02-13:39:50", Rate:  "70%", Credit:  "89" },
        { Element: "9875-2", Channel: "view",            Time: "2020/03/03-09:22:37", Rate:  "80%", Credit: "101" },
        { Element: "9875-3", Channel: "view",            Time: "2020/03/03-09:43:05", Rate:  "80%", Credit: "101" },
        { Element: "7788-3", Channel: "email open",      Time: "2020/03/03-15:48:26", Rate:  "80%", Credit: "101" },
        { Element: "7890-1", Channel: "view",            Time: "2020/03/03-22:38:13", Rate:  "80%", Credit: "101" },
        { Element: "9875-4", Channel: "click",           Time: "2020/03/04-17:11:56", Rate:  "90%", Credit: "114" },
        { Element: "5678-1", Channel: "organic search",  Time: "2020/03/05-08:31:48", Rate: "100%", Credit: "127" },
        { Element: "5678-2", Channel: "internal search", Time: "2020/03/05-08:33:12", Rate: "100%", Credit: "127" }
    }
    
For these examples, giving helper 2 access to the channel value is not needed. However, more advanced attribution algorithms treat different channels differently. For example, some might may assign different weights to different channels or use different decay rates for them (clicks are more valuable than views; ad views that happened last week likely contributed less than ad clicks from the previous week). After it completes its attribution, helper 2 would prepare to send the following data to helper 1:

>**Helper 2 Data to be sent to Helper 1**

    Attribution: {
        { Element: "5678-1", Credit: "127" },
        { Element: "5678-2", Credit: "127" },
        { Element: "7788-1", Credit:  "63" },
        { Element: "7788-2", Credit:  "76" },
        { Element: "7788-3", Credit: "101" },
        { Element: "9875-1", Credit:  "89" },
        { Element: "9875-2", Credit: "101" },
        { Element: "9875-3", Credit: "101" },
        { Element: "9875-4", Credit: "114" },
        { Element: "9890-1", Credit: "101" },
        { ... (values for all other conversions, which could be intermixed with those for the example conversion)
    }
Once helper 1 gets this data, it finds the associated data elements and gives them the computed credit. It then aggregates this by channel and by the sub-channels (ad IDs, campaign IDs, email IDs, search terms, etc.) specific to each channel.

>**Helper 1 Intermediate results: Attribution at Interaction Level**

    IntermediateAttribution: {
        { Channel: "view",            AdID: "abc",    CampaignID: "xyz", Credit:  "89" },
        { Channel: "view",            AdID: "def",    CampaignID: "pqr", Credit: "101" },
        { Channel: "view",            AdID: "abc",    CampaignID: "xyz", Credit: "101" },
        { Channel: "click",           AdID: "mno",    CampaignID: "stu", Credit: "114" },
        { Channel: "view",            AdID: "jkl",    CampaignID: "pqr", Credit: "101" },
        { Channel: "organic search",  SearchTerms: "widget1",            Credit: "127" },
        { Channel: "internal search", SearchTerms: "doohickey3",         Credit: "127" },
        { Channel: "email send",      EmailID: "123", CampaignID: "pqr", Credit:  "63" },
        { Channel: "email open",      EmailID: "123", CampaignID: "pqr", Credit:  "76" },
        { Channel: "email open",      EmailID: "123", CampaignID: "pqr", Credit: "101" }
    }
    
which when grouped by channel and sub-channel yields:

>**Attribution at Channel/Sub-channel**

    FinalAttribution: {
        { Channel: "view",            Credit: "392" }, // 89+101+101+101
        { Channel: "click",           Credit: "114" },
        { Channel: "organic search",  Credit: "127" },
        { Channel: "internal search", Credit: "127" },
        { Channel: "email send",      Credit:  "63" },
        { Channel: "email open",      Credit: "177" },
 
        { SearchTerms: "widget1",     Credit: "127" },
        { SearchTerms: "doohickey3",  Credit: "127" },
 
        { AdID: "abc",                Credit: "190" },
        { AdID: "def",                Credit: "101" },
        { AdID: "mno",                Credit: "114" },
        { AdID: "jkl",                Credit: "101" },
        { EmailID: "123",             Credit: "240" },
 
        { CampaignID: "xyz",          Credit: "190" },
        { CampaignID: "stu",          Credit: "114" },
        { CampaignID: "pqr",          Credit: "442" }
    }
    
This data is then combined with the identical channels and sub-channels from all other conversions known to helper 1, keeping track of the number of conversions that contributed to each value. All values that did not have contributions from a sufficient number of conversions are dropped. Noise would be introduced as described in the original Google proposal to each remaining value. Then, helper 1 can return this aggregate data to the advertiser, which it would combine with the aggregate data returned by helper 2 (for odd numbered conversion IDs).

## Channels
If it is necessary to include a predefined list of channels that are supported, the list should include:

* ad view
* ad click
* email send
* email open
* email click
* call center
* in-store
* push notification
* social post
* onsite offers - promotions shown to the user on the advertiser's website
* internal search (performed a search on the advertiser's website)
* organic search (used a search engine and clicked on a non-ad result to come to the advertiser's site; benefits from SEO investments)

With the exception of ad views, the advertiser has the ability to tie all the above interactions to the user. In fact for this multi-channel attribution to work, they must have tied all channels they care about except ad clicks and ad views to the user, since they (or their service providers) will need to generate the interaction list for each channel when the user converts.

Note that the actions of organic search and internal search are actions that both occurred in one of the browsers, so instead of requiring the Advertiser to capture these details and create this report later, perhaps the browser could somehow be notified about these searches and treat them somewhat like ads for this reporting, including them in the browser reports? Note, when the user performs multiple searches, typically, we would only want the final search to receive credit, because it is likely they continued their searching because the first searches didn't return the results they were looking for and thus were not valuable.

## Threat Models
There is probably some risk of storing the timestamps when each ad was displayed in one of the files as this may help associate the user with the ad instances. However, only one of the helpers has access to the timestamps for any given conversion, and it does not have the ad IDs or campaign IDs for any of those ads. Still we could reduce its knowledge by having the browser remove the conversion timestamp from the file and record the interaction times relative to the conversion time, such as the number of seconds (or minutes or hours) before the time of the conversion (the lower the resolution of this, the harder it will be for the attribution algorithm to get the interactions from various sources into the correct order, which is often important for identifying the first or last interaction before a conversion).

## Limitations
### Mobile Apps
I am not including details about how this might work with mobile apps, in regards to either the ads displayed in those apps or conversions that occur within them, unless that app has an embedded web browser that can be tied to the user's other browsers. However, I believe this is a solvable problem that can be addressed if this proposal is adopted.

### Offline Conversions
This solution only supports conversions that happen within the web browser rather than other methods such as call-center or in-store purchases, because I do not know how to inform the browser to send in the appropriate list of viewed/clicked ads in a privacy preserving manner when the conversion does not occur in the browser.

### Conversion Types
I don't document support for multiple conversion types here, but I can envision extensions that could support that functionality if this concept is adopted.

### Publisher Contributions
It is often valuable to know the publisher sites where ads were most effective in leading to conversions, so they can be favored for future advertising. It would be a simple extension to include the domain of each publisher for each listed ad and compute attributions separately for each publisher. Again publisher data would only be returned by the helpers if a sufficient number of conversions were attributed to that publisher.

## Advanced Attribution Algorithms
One of the services that Adobe provides to its customers is support for advanced attribution algorithms that apply machine learning to compute the optimal values for the weighting and decay rate values. This requires recomputing the attribution for a large set of conversions (typically a full month), hundreds or thousands of times and adjusting the weight/decay values until they serve as good predictors of the conversion values. These algorithms don't need access to the conversion IDs, ad IDs or campaign IDs, but do need access to the channel of each interaction and the time of the interaction relative to the conversion, along with the conversion value.  The data for our sample conversion would look something like:

>**Helper 3 Dataset**

    Interactions: {
        { Channel: "email send",      MinutesBeforeConversion: "7515" },
        { Channel: "email open",      MinutesBeforeConversion: "5671" },
        { Channel: "view",            MinutesBeforeConversion: "4017" },
        { Channel: "view",            MinutesBeforeConversion: "2834" },
        { Channel: "view",            MinutesBeforeConversion: "2813" },
        { Channel: "email open",      MinutesBeforeConversion: "2448" },
        { Channel: "view",            MinutesBeforeConversion: "2038" },
        { Channel: "click",           MinutesBeforeConversion:  "925" },
        { Channel: "organic search",  MinutesBeforeConversion:    "5" },
        { Channel: "internal search", MinutesBeforeConversion:    "3" }
    },
    Conversion-value: "1000"
    
The first and second helpers could send just this data to a third helper service for a full month, which could compute the optimal weighting and decay values and return them to the advertiser. The advertiser would use these during the subsequent day or week until the process was repeated to compute new values. Could service providers such as Adobe provide proprietary implementations for this third helper if they promised that the helper would not to attempt to associate the data shared by helpers 1/2 with any data from the original datasets or to identify specific users in that dataset? If thought necessary to prevent re-identification, helpers 1/2 could add a small amount of noise to the conversion values and/or MinutesBeforeConversion values. Reducing timing resolution from minutes to hours would be another alternative to make the association harder.
