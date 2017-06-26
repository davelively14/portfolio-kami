---
layout: post
title: "Capstone - Part 1"
---
## Background

There were several reasons why I chose [Bloc](https://www.bloc.io/) over other bootcamps, but one of the more attractive aspects was that I was able to go at my own pace. I ran at an accelerated clip through the early lessons with which I was already familiar. That left me significantly more time to dedicate towards my capstone project for the Web Development portion of my [Software Development track](https://www.bloc.io/software-developer-track).

The idea for my capstone came from my wife. After one of those senseless, online bullying induced suicides, my wife suggested that I develop an app where parents could monitor their loved one’s social media public feeds for any signs of bullying or outward displays of suicide ideation.

In order to limit the scope of what promised to be a significant undertaking for a single developer, I chose to narrow the focus to a single social media site initially. Given its robust API and general ease of use, I chose [Twitter](https://dev.twitter.com/). Additionally, I chose to forego a mailer or push notification system and focus initially on an on-demand tracking system. Architecturally, however, I designed and implemented the system so that we can easily add additional social media sites and a mailer system as time and resources permit.

## The Stack

When it came to picking the tech stack for this project, my mentor, [Ryan Milstead](https://www.linkedin.com/in/ryanmilstead/), recommended that I select technologies that I wanted to work with instead of being constrained by what we had already used. There wasn’t anything particularly wrong with Rails, JQuery, or AngularJS, but with an open sandbox and my pick of tools, I selected my favorites: Elixir on the backend, with Phoenix providing the web interface layer, and React with Redux for the frontend demo.

### Elixir and Phoenix

Full disclosure: [Eilxir](https://elixir-lang.org/) - a functional language built on top of Erlang and compiles into BEAM - is my favorite language, so I’m a little biased. While Python or any number of other languages might have been more performant in terms of parsing and analyzing the collected data, few can compete with Elixir’s scalability, fault tolerance, and maintainability.

In order to develop a wider variety of skills in this academic setting, I chose to use two different Phoenix endpoints for my client interaction layer: a JSON API for handling all things related to the user profile and authentication, and the [famously](http://www.phoenixframework.org/blog/the-road-to-2-million-websocket-connections) efficient Phoenix Channels to manage websocket connections that exposes social media monitoring functionality to the user.

In order to collect the data, I would utilize Elixir’s built in Erlang OTP abstractions to supervise the fetching, parsing, and serving results from Twitter.

### Living on the Edge - Phoenix 1.3.0-rc

Since this is an app I’m building in order to further my education, I opted to venture out on the bleeding edge and use Phoenix 1.3.0-rc. Don’t let the what seems like a minor version change fool you. While there are no real breaking changes or significant new features being introduced in 1.3 there is a seismic shift in how a developer approaches a Phoenix project due to the changes in the directory structure and code generators. Chris McCord provides an overview and insight into the changes in the [Lonestar Elixir keynote address](https://www.youtube.com/watch?v=tMO28ar0lW8), his [Elixir Forum post](https://elixirforum.com/t/phoenix-v1-3-0-rc-0-released/3947) and his more recent [ElixirConf EU keynote](https://www.youtube.com/watch?v=pfFpIjFOL-I), but I found it more easy to understand the advantages once I began implementing it in a project.

While the Phoenix teams does emphasize that these changes are suggestions for how to organize your code, for the purposes of this project I chose to treat it as gospel. For me, these are the three big takeaways from this new version;
While Phoenix has only ever been the web layer for your Elixir app, the new directory structure makes that abundantly clear. Everything Phoenix related is all neatly packed away within the `lib/my_app` directory, including the `web` directory.

Models are gone. Instead, developers create systems with a `Context`, which is simply a module that acts as a boundary for the underlying system. Different systems will often use their own schemas to reference a common table in the database, but only retrieve and update the columns for which they’re responsible. More on this later, as its utility becomes more apparent in an example.

Assets no longer clog up the root directory. All of your external assets are stored in the `/assets` directory. Since I was using Webpack for this project, this was a particular pain for me initially, but ultimately it makes the project significantly easier to navigate. If you’re interested, [here’s how I setup and configured](https://github.com/davelively14/configs/blob/master/phoenix_1.3_react_redux.md) my Phoenix 1.3 app to work with Webpack 2, React, Redux, and React Router.

Ultimately, Phoenix 1.3 provides a forcing mechanism to design with intent. You have to think about what you’re going to do before you do it. I struggled early with the new structure, but ultimately these guidelines lead to better app development with clear divisions of responsibility.

## Designing the System - the stuff dreams are made of

A quick note on the nomenclature I chose. In keeping with the mythical bird theme that Phoenix started, I chose to name several of my systems after characters from the classic 1941 film, [The Maltese Falcon](https://en.wikipedia.org/wiki/The_Maltese_Falcon_(1941_film)).  Sam *Spade* interacts with the *Client*, analyzes the case, and dispatches his partner, Miles *Archer*, to gather information on the quarry.

![Figure 1](/img/capstone/overview.png)
Figure 1. Client and Spade systems expose their API to user requests via Phoenix web endpoints. Both systems are contained within the `web` directory and are accessible to the user via JSON and Channel endpoints, respectively. `SpadeInspector` and `Archer` are Phoenix 1.3 styled systems that are accessible by the via their `Context` of the same name.

In my app’s basic form, the `Client` system handles all the requests relating to a particular user, such as creating a profile, editing that profile, and authorizing access via tokens. The `Spade` system is where the significant action happens. Via websocket connection, the user requests existing data, which uses its schemas to pull existing data from the database. When new data is requested, `Spade` will utilize`SpadeInspector` to manage deployment of `Archer` to fetch, parse and return results. Upon receipt of new results, `SpadeInspector` will evaluate those results, store them, and asynchronously return them to the user.

Fortunately, we made a few improvements on the movie, so if/when Archer dies, he’ll be seamlessly revived and put back into action thanks to Elixir’s awesome reliability.

So what does this project look like in Phoenix 1.3?

<pre>
flatfoot
|-- assets
|-- config
|-- <b>lib/flatfoot</b>
|   |-- <b>archer</b>
|   |-- <b>clients</b>
|   |-- data
|   |-- shared
|   |-- <b>spade</b>
|   |-- <b>spade_inspector</b>
|   |-- <b>web</b>
|   |-- application.ex
|   |-- repo.ex
|
|-- priv
|-- test
|-- .eslintrc.js
|-- .gitignore
|-- .travis.yml
|-- README.md
|-- mix.exs
|-- mix.lock
</pre>

If you’re used to running a third party bundler like webpack with Phoenix 1.2 and earlier, you’ll quickly notice how uncluttered the root directory is.  All of that is packed away in the `assets` directory, and everything in our Phoenix app is packed away neatly within the `lib/flatfoot` directory - including `/web`.

Each system has it’s own directory, with a matching named `Context` file. For my app, I broke slightly with the out of the box Phoenix 1.3.0-rc directory convention. In order to avoid cluttering each system’s root directory with schema, I nested all of the schemas within the `/schema` directory:

<pre>
|-- spade
    |-- schema
    |   |-- backend.ex
    |   |-- suspect.ex
    |   |-- suspect_account.ex
    |   |-- user.ex
    |   |-- ward.ex
    |   |-- ward_account.ex
    |   |-- ward_result.ex
    |   |-- watchlist.ex
    |
    |-- spade.ex
</pre>

It’s important to note that these systems can be more than just a single `Context` file and schema. Any Elixir modules affecting the system can - and most likely should - be included within the boundary. In my `Archer` context (see directory tree below), I included an `archer/otp` directory to house my `ArcherSupervisor`, `Archer.Server`, and `/backends`, which contains my task modules.

<pre>
|-- archer
    |-- otp
    |   |-- backends
    |   |   |-- twitter.ex
    |   |
    |   |-- archer_supervisor.ex
    |   |-- server.ex
    |
    |-- schema
    |   |-- backend.ex
    |
    |-- archer.ex
</pre>

The key to maintaining the boundary here, is that any interactivity with the server or backend schema goes through the `archer.ex` context. For example, `Flatfoot.Archer.fetch_data/1` exposes the `Flatfoot.Archer.Server.fetch_data/1` function:

```elixir
##########
# Server #
##########

alias Flatfoot.Archer.Server

def fetch_data(configs) do
  Server.fetch_data(configs)
end
```

## Visualizing the Data Flow for a Request


![Figure 2](/img/capstone/data_flow.png)
Figure 2. Data flow overview

While much of the interaction between the user and backend will be via JSON and websocket to interact with existing data, the purpose of the app is to get information from social media sites, which is what we focus on for this section.

Figure 2 is pretty busy, so we’ll break down this flow of data step by step.



Step 1. An authenticated user will pushe a request to the SpadeChannel websocket via a `fetch_new_ward_results` message with a `{“id”: ward_id}` payload. `Wards` are individuals the user wishes to keep tabs on. Each `Ward` will have none, one, or many `WardAccounts`, which are individual accounts for various networks (i.e. handles on Twitter, usernames for Facebook).



Step 2. Upon authentication and parameter check, `SpadeChannel` will send send a request to the `SpadeInspector.Server` via the `SpadeInspector.Server.fetch_update(ward_id)` function call. The server starts on app launch, so it’s already running. The server will handle the call internally and retrieve the correct `Ward`. For each `WardAccount` of that `Ward`, the server will build an `mfa` map with instructions for processing and then assign all of the `mfa` maps to a list named `configs`.



Step 3. `SpadeInspector.Server` sends the list of `mfa` maps to the `Archer` system via the `Archer.Server.fetch_data(configs)` function call.
