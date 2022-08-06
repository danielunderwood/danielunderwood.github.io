---
title: Projects
--- 
## <a href="https://github.com/danielunderwood/feeds"><i class="fab fa-github"></i></a> Feeds

I consume a fair bit of news in the form of RSS feeds. Unfortunately, there are a number of sources that don't publish in the
form of RSS feeds. This project was started with [CISA's exploited vulnerabilities catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog),
where emails are sent out announcing that a vulnerability was added to the list, but you have to go to the table and sort it
to actually find out what's new.

I realized that I could deploy a feed that parsed the catalog fairly quickly with [Cloudflare Workers](https://workers.cloudflare.com/)
(_disclaimer: I'm a Cloudflare shareholder_), so I put together the project one afternoon and plan to add additional sources
as needed (or feel free to PR your own!).

### Currently Supported Sources
- <a href="https://feeds.danielunderwood.dev/exploited-vulns/feed.xml"><i class="fa fa-rss"></i></a> CISA Exploited Vulnerabilities Catalog
