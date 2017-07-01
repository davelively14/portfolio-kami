---
layout: post
title: Bloccit
thumbnail-path: "img/bloccit-1.png"
short-description: An SEO tool to check accuracy of local listings for a given company.

---

{:.center}
![]({{ site.baseurl }}/img/bloccit-1.png)

![](https://github.com/favicon.ico) &nbsp;&nbsp;[View on GitHub](https://github.com/davelively14/bloc-bloccit) <br>
<img src="https://image.flaticon.com/icons/png/128/12/12195.png" width="32"> &nbsp;&nbsp;[View Demo](https://immense-brook-70799.herokuapp.com/)

## Explanation

As part of the Foundation Backend Web Development portion of Bloc's [Software Developer Track](https://www.bloc.io/software-developer-track), we were tasked to develop an application that works similar to Spotify.

<!-- A key aspect of any SEO campaign involves a dedicated approach to improve local search rankings. Many of the top factors [listed by Moz](https://moz.com/local-search-ranking-factors) that lead to a negative rating are with respect to the accuracy of a business' information on local listing sites. [Nebo Agency](http://www.neboagency.com/), an Atlanta based online marketing agency, was seeking a solution for automating the process of checking local listings on a series of local listing sites. With no solution readily available to agencies, I was asked to assist with the creation of an application that would aggregate search results, rank them, and provide links for the SEO team to make the necessary changes.

Full disclosure, my [wife](http://www.neboagency.com/about/people/#person-57) is the Associate Director of SEO at Nebo. -->

## Problem

<!-- Initially, the SEO team at Nebo had to check local listings for each their clients on 11 separate sites, one at a time. While there were some services already available, like [Moz local](https://moz.com/local), they all required some sort of subscription service where the third party would manage all of the listings. In addition to being cost prohibitive and more intensive than necessary, if you cancel the subscription for a particular listing then the listings would revert back to their previous condition. All Nebo really needed was to see the listings for all of their clients so they could fix inaccurate information.

Specifically, Nebo was looking for a solution that addressed these needs:
- Add a series of listings to a project.
- Retrieve local listings from ten identified sources.
- Display each result from those local listings with rankings for accuracy and a link to each result page.
- Maintain previous results.
- Present in a way that encourages collaboration among the team. -->

## Solution

<!-- For my technology stack, I chose [PostgreSQL](https://www.postgresql.org/) for my database layer, [Elixir](http://elixir-lang.org/) with it's Erlang underbelly and OTP architecture for my backend, the [Phoenix Framework](http://www.phoenixframework.org/) for routing, rendering static pages, and channels, and straight JavaScript for client side. While I could have gone a variety of ways, a functional language like Elixir, with it's [cheap processes](http://erlang.org/doc/efficiency_guide/processes.html) and concurrent nature, would allow me to scrape websites or pull from API's asynchronously, collect them, and return to the user with relative ease. Additionally, since collaboration was a priority, Phoenix's Channels [perform considerably better](https://dockyard.com/blog/2016/08/09/phoenix-channels-vs-rails-action-cable) than Rail's Action Cable. -->

## Conclusion

<!-- While _Locorum_ is still in development, the SEO team at Nebo is currently using the app as intended. We are continually working to add new features, such as the ability to ignore some search results and persist results even in situations where there are none. In order to more effectively work with data on the client side, I am refactoring JavaScript to React with Redux. While standard JavaScript has served us well, we will need more effective state management in order to implement the more complex solutions. -->
