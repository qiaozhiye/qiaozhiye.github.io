---
categories:
- resource
date: "2023-11-05"
slug: matching-iss-voting-with-crsp
tags:
- ISS
- CRSP
- shareholder voting
title: "Match ISS Voting Analytics with CRSP Mutual Funds"
headline: Here is a summary of the procedure, built on Peter Iliev's note, for matching mutual funds between ISS Voting Analytics and CRSP databases.
authors:
  - "Qiaozhi Ye"
authorURLs:
  - "https://www.yegeorge.com"

---

In Appendix B of [Sulaeman and Ye (2023)]({{< ref "race-and-voting.md" >}}), we perform the following procedure to link ISS mutual funds (FundID) with CRSP mutual funds (CRSP\_PORTNO).

As described in [Peter Iliev's note](https://bpb-us-e1.wpmucdn.com/sites.psu.edu/dist/b/169215/files/2023/08/voting-link-note-v2.pdf), each proxy voting record in the ISS data can be linked to the original SEC Form N-PX using the reference identifier (NPXFileID). From the SEC's N-PX file, we obtain a list of fund class tickers (TICKER) associated with the registered management investment company on the filing date. Because the CRSP Mutual Fund Summary data provide a direct linkage between the fund class tickers (TICKER) and the fund portfolio identifiers (CRSP\_PORTNO), we are able to map FundID from ISS to CRSP\_PORTNO from CRSP by TICKER in each quarter.

We observe that, in most cases (88\% in our exercise), a FundID in a quarter is matched with multiple CRSP\_PORTNOs, because a N-PX file typically refers to multiple funds under the same investment management company. For each FundID, we identify the most probable CRSP\_PORTNO via matching the fund name between the two databases, using both Jaro-Winkler and Levenshtein Distance name-matching [algorithms](https://cran.r-project.org/web/packages/stringdist/index.html). We retain the pairs of FundID-CRSP\_PORTNO with the minimum name distance according to the two algorithms and further require the distance to be less than 0.3 for Jaro-Winkler and 10 for Levenshtein Distance. In about 72\% of the FundID-CRSP\_PORTNO pairs, Jaro-Winkler or Levenshtein Distance reports a perfect match between the ISS and the CRSP fund names. For the remaining 28\% of the cases where fund names are not exactly matched, we manually verify the accuracy of the mappings. As our name-matching methodology tightens the links between FundID and CRSP\_PORTNO within an investment management company in a quarter, it performs better than a general, unconditional matching using a universe of fund names from the two databases.

Suggested citation:
Sulaeman, J. and Ye, Q.(2023). Who do you vote for? Same-race voting preferences in director elections. European Corporate Governance Institute – Finance Working Paper No. 872/2023
