---
layout: post
title: Locorum
thumbnail-path: "img/locorum-1.png"
short-description: An SEO tool to check accuracy of local listings for a given company.

---

{:.center}
![]({{ site.baseurl }}/img/locorum-1.png)

![](https://github.com/favicon.ico) &nbsp;&nbsp;[View on GitHub](https://github.com/davelively14/locorum) <br>
<img src="https://image.flaticon.com/icons/png/128/12/12195.png" width="32"> &nbsp;&nbsp;[View Demo](https://boiling-beach-47326.herokuapp.com)

## Explanation

A key aspect of any SEO campaign involves a dedicated approach to improve local search rankings. Many of the top factors [listed by Moz](https://moz.com/local-search-ranking-factors) that lead to a negative rating are with respect to the accuracy of a business' information on local listing sites. [Nebo Agency](http://www.neboagency.com/), an Atlanta based online marketing agency, was seeking a solution for automating the process of checking local listings on a series of local listing sites. With no solution readily available to agencies, I was asked to assist with the creation of an application that would aggregate search results, rank them, and provide links for the SEO team to make the necessary changes.

Full disclosure, my [wife](http://www.neboagency.com/about/people/#person-57) is the Associate Director of SEO at Nebo.

## Problem

Initially, the SEO team at Nebo had to check local listings for each their clients on 11 separate sites, one at a time. While there were some services already available, like [Moz local](https://moz.com/local), they all required some sort of subscription service where the third party would manage all of the listings. In addition to being cost prohibitive and more intensive than necessary, if you cancel the subscription for a particular listing then the listings would revert back to their previous condition. All Nebo really needed was to see the listings for all of their clients so they could fix inaccurate information.

Specifically, Nebo was looking for a solution that addressed these needs:
- Add a series of listings to a project.
- Retrieve local listings from ten identified sources.
- Display each result from those local listings with rankings for accuracy and a link to each result page.
- Maintain previous results.
- Present in a way that encourages collaboration among the team.

## Solution

For my technology stack, I chose [PostgreSQL](https://www.postgresql.org/) for my database layer, [Elixir](http://elixir-lang.org/) with it's Erlang underbelly and OTP architecture for my backend, the [Phoenix Framework](http://www.phoenixframework.org/) for routing, rendering static pages, and channels, and straight JavaScript for client side. While I could have gone a variety of ways, a functional language like Elixir, with it's [cheap processes](http://erlang.org/doc/efficiency_guide/processes.html) and concurrent nature, would allow me to scrape websites or pull from API's asynchronously, collect them, and return to the user with relative ease. Additionally, since collaboration was a priority, Phoenix's Channels [perform considerably better](https://dockyard.com/blog/2016/08/09/phoenix-channels-vs-rails-action-cable) than Rail's Action Cable.


#### Database Layer
<img align="right" src="/img/locorum-3.png">
We can use the database schema to provide an overview for how the application works to solves the issues. A `User` creates an account, setting their name, username, and a password via a bcrypt hash. That user is then able to create a `Project` and then add any number of listings as an individual `Search`, which would all belong to the `Project`. Every time the user uses the app to run a search, a `ResultCollection` is created. Each `Backend` then asynchronously generates any number of `Result` entries, which are then returned to the user.

#### Adding a series of listings
<img align="left" src="/img/locorum-2.png" width="300">
SEO specialists at Nebo will download their client's information as a CSV from [Google My Business](https://support.google.com/business/answer/3478406?hl=en), which they use as a basis to conduct searches. Once a user creates a project, they are presented with the option to either `Create New Search`, which displays a modal with an HTML form for manual creation, or `Upload CSV`. If the user opts for CSV, they can upload a Google My Business CSV and automatically generate any number of searches that will be associated with the current `Project`. We use an Elixir controller, `csv_controller.ex`, to parse the CSV file, persist the results to the database, and associate it with the current `Project`.

#### Retrieving local listings

Once searches are added, the user may then navigate to the results page, which is a [Phoenix Channel](http://www.phoenixframework.org/docs/channels). While we could have solved this particular problem without using a websocket, Nebo's final problem listed above was to present these results with the ability to collaborate. Future versions will include more collaborative features, such as live chat and real time comments.

<img align="right" src="/img/locorum-4.png" width="450">
When a user conducts a search, Elixir's concurrency advantages and efficient supervision trees quickly becomes evident. We spin up a supervisor that manages `BackendSys`, a module that will asynchronously starts every backend module. Each module will either use an API from one of the backends to pull and parse JSON data, or scrape and parse HTML. Once each backend module parses the results, they send them back to the channel, where the clients are listening via JavaScript and render the results in the overview pane for each search.

#### Displaying each result

<img align="left" src="/img/locorum-5.png" width="300">
Nebo had specific requirements for displaying results. First, they wanted an accuracy rating for each search. In order to achieve this, we created a function in the `Locorum.BackendSys.Helpers` module (see below). We use the built in Jaro distance function to compare each element of the result (business name, address, city, state, zip, and phone) to the listing provided by the user and return the lowest score. We make a few exceptions in order to capture the importance of the zip and phone. If the zip code is incorrect, the maximum score is a 20. If the phone is incorrect, the maximum score is a 50.

{% highlight elixir %}
def rate_results(results, query) do
  address = single_address(query.address1, query.address2)

  for result <- results do
    rating =
      %{biz: rate_same(result.biz, query.biz), address: rate_same(result.address, address),
        city: rate_same(result.city, query.city), state: rate_same(result.state, query.state),
        phone: rate_same(phonify(result.phone), phonify(query.phone))}
      |> return_lowest

    rating =
      cond do
        rating < 0.2 ->
          rating
        result.zip && (result.zip != query.zip) ->
          0.2
        rating < 0.5 ->
          rating
        result.phone && (phonify(result.phone) != phonify(query.phone)) ->
          0.5
        true ->
          rating
      end

    Map.put(result, :rating, round(rating * 100))
  end
end
{% endhighlight %}

The other requirement for displaying results is to provide a link to result on the local listing site. During fetch and parsing, each backend will provide a link (`View at Source`) to the page where the SEO specialist can request improvements.

#### Maintain previous results

<img align="right" src="/img/locorum-6.png" width="300">
An additional feature requested by the team at Nebo was to maintain a collection of previous results in order to track improvements and rankings. All previous results are available. This is a good time to explain how we leveraged Elixir's built in [OTP](http://learningelixir.joekain.com/designing-with-otp-applications-in-elixir/) as middleware to speed up requests for data. Whenever a project channel is created, we start a [GenServer](https://hexdocs.pm/elixir/GenServer.html) that pulls all the data for all the searches from the underlying database for a given project.

In order to maintain continuity between data on the GenServer and the database, only the GenServer can write to the database. All new search results are sent to the server, which then persists them on the database. When the user requests older search results, the server is able to return the most up to date results nearly instantaneously. While the time savings may be negligible here, this will allow us to scale quite readily, as read requests to the database are few and far between.

<!-- ## Results

While _Locorum_ is still in development, the SEO team at Nebo is currently using the app as intended. We are continually working to add new features, such as the ability to ignore some search results and persist results even in situations where there are none. In order to more effectively work with data on the client side, I am refactoring JavaScript to React with Redux. While standard JavaScript has served us well, we will need more effective state management in order to implement the more complex solutions. -->

## Conclusion

While _Locorum_ is still in development, the SEO team at Nebo is currently using the app as intended. We are continually working to add new features, such as the ability to ignore some search results and persist results even in situations where there are none. In order to more effectively work with data on the client side, I am refactoring JavaScript to React with Redux. While standard JavaScript has served us well, we will need more effective state management in order to implement the more complex solutions.
