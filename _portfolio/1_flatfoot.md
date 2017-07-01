---
layout: post
title: Flatfoot
thumbnail-path: "img/flatfoot-1.png"
short-description: Monitors and reports cyberbullying and cries for help

---

{:.center}
![]({{ site.baseurl }}/img/flatfoot-1.png)

![](https://github.com/favicon.ico) &nbsp;&nbsp;[View on GitHub](https://github.com/davelively14/flatfoot) <br>
<img src="https://image.flaticon.com/icons/png/128/12/12195.png" width="32"> &nbsp;&nbsp;[View Demo](https://obscure-reef-78695.herokuapp.com/)

## Explanation

There were several reasons why I chose [Bloc](https://www.bloc.io/) over other bootcamps, but one of the more attractive aspects was that I was able to go at my own pace. I ran at an accelerated clip through the early lessons with which I was already familiar, leaving me with significantly more time to dedicate towards the Web Development capstone project of the [Software Development track](https://www.bloc.io/software-developer-track).

The idea for this project came from my [wife](http://www.neboagency.com/about/people/#person-57). After one of those senseless, cyberbullying induced suicides, my wife though of an app where parents could monitor their loved one’s social media public feeds for any signs of bullying or outward displays of suicide ideation.

In order to limit the scope of what promised to be a significant undertaking for a lone developer, I chose to collect on a single social media site initially. Given its robust API and general ease of use, I chose [Twitter](https://dev.twitter.com/). Additionally, I opted to initially forego implementing a mailer or push notification system and instead focus on building an on-demand tracking system. Architecturally, however, I designed the app so that functionality could be included in future versions.

## Problem

As a parent, it can be difficult to monitor your teen's social media accounts. Balancing safety and privacy concerns, the sheer volume of content, and high likelihood of missing a post or two are all significant challenges. Having a one stop shop that a parent or loved one could use to screen social media content, curated and presented with potentially troubling content first, could prove a valuable tool.

## Solution

For my technology stack, I chose [PostgreSQL](https://www.postgresql.org/) for my database layer, [Elixir](http://elixir-lang.org/) with it's Erlang underbelly and OTP architecture for my backend, [Phoenix 1.3](http://www.phoenixframework.org/) as the web interface layer, and [React](https://facebook.github.io/react/) with [Redux](http://redux.js.org/) for the client side. While Python or other languages might have been more performant in terms of parsing and analyzing the collected data, few can compete with Elixir’s out of the box scalability, fault tolerance, and maintainability as a whole.

### Overview

_Flatfoot_ operates as a single-page application (SPA) that interacts with the backend via a JSON API for user authentication and profile management, and websocket for all things related to monitoring social media accounts.

#### Nomenclature

![Maltese Falcon, 1941](http://www.filminamerica.com/Movies/TheMalteseFalcon/falcon06.jpg)

In keeping with the Phoenix mythical bird theme, I named my systems after characters from the classic 1941 film, [The Maltese Falcon](https://en.wikipedia.org/wiki/The_Maltese_Falcon_(1941_film)). The relevant part of the premise is rather simple: A prospective **Client** approaches Sam **Spade** with an interesting case. Spade takes the case and dispatches his partner, Miles **Archer**, to gather information on the case and report back, but ultimately it's up to Spade to solve the mystery. Fortunately, we made a few improvements on the movie, so if Archer just happens to die, he’ll be seamlessly revived and put back into action thanks to the magic of OTP.

I settled on the name 'Flatfoot' for the app. I realize that's more a beat cop than a private investigator like Spade, but the prevailing slang for a PI at the time doesn't seem appropriate in a modern context.

#### Database Layer
<img align="right" src="/img/flatfoot-2.png">
We can use the database schema to provide an overview for how _Flatfoot_ works. A `User` creates an account, setting their name, username, and a password via a bcrypt hash. That user is then able to create a `Project` and then add any number of listings as an individual `Search`, which would all belong to the `Project`. Every time the user uses the app to run a search, a `ResultCollection` is created. Each `Backend` then asynchronously generates any number of `Result` entries, which are then returned to the user.

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

### The Frontend

[`react-router`](https://github.com/ReactTraining/react-router/tree/master/packages/react-router) to manage


## Conclusion

While _Locorum_ is still in development, the SEO team at Nebo is currently using the app as intended. We are continually working to add new features, such as the ability to ignore some search results and persist results even in situations where there are none. In order to more effectively work with data on the client side, I am refactoring JavaScript to React with Redux. While standard JavaScript has served us well, we will need more effective state management in order to implement the more complex solutions.
