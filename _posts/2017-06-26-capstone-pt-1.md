---
layout: post
title: "Capstone - Phoenix 1.3, pt 1"
---
## Background

There were several reasons why I chose [Bloc](https://www.bloc.io/) over other bootcamps, but one of the more attractive aspects was that I was able to go at my own pace. I ran at an accelerated clip through the early lessons with which I was already familiar. That left me significantly more time to dedicate towards my capstone project for the Web Development portion of my [Software Development track](https://www.bloc.io/software-developer-track).

The idea for my capstone came from my wife. After one of those senseless, online bullying induced suicides, my wife suggested that I develop an app where parents could monitor their loved one’s social media public feeds for any signs of bullying or outward displays of suicide ideation.

In order to limit the scope of what promised to be a significant undertaking for a lone developer, I chose to narrow the focus to a single social media site initially. Given its robust API and general ease of use, I chose [Twitter](https://dev.twitter.com/). Additionally, I chose to forego implementing a mailer or push notification system and instead focus initially on an on-demand tracking system. Architecturally, however, I designed the app so that we can easily add additional social media sites and a mailer system as time and resources permit.

## The Stack

When it came to picking the tech stack for this project, my mentor, [Ryan Milstead](https://www.linkedin.com/in/ryanmilstead/), recommended that I select technologies that I wanted to work with instead of being constrained by what we had already used. There wasn’t anything particularly wrong with Rails, JQuery, or AngularJS, but with an open sandbox and my pick of tools, I selected my favorites: [Eilxir](https://elixir-lang.org/) on the backend, with [Phoenix](http://www.phoenixframework.org/) providing the web interface layer, and [React](https://facebook.github.io/react/) with [Redux](http://redux.js.org/) for the frontend demo.

### Elixir and Phoenix

Full disclosure: Elixir - a functional language built on top of Erlang and compiles into BEAM - is my favorite language, so I’m a little biased. While Python or any number of other languages might have been more performant in terms of parsing and analyzing the collected data, few can compete with Elixir’s out of the box scalability, fault tolerance, and maintainability.

In order to develop a wider variety of skills in this academic setting, I chose to use two different Phoenix endpoints for my user interaction layer. First, a JSON API for handling all things related to the user profile, settings, and authentication. Second, I would use a [famously](http://www.phoenixframework.org/blog/the-road-to-2-million-websocket-connections) efficient Phoenix Channel to manage websocket connections that exposes social media monitoring functionality to the user.

In order to collect the data, I would utilize Elixir’s built in Erlang OTP abstractions to supervise the fetching, parsing, and serving results from Twitter.

### Living on the Edge

Since this is an app I’m building in order to further my education, I opted to venture out on the bleeding edge and use Phoenix 1.3.0-rc. Don’t let the what seems like a minor version change fool you. While there are no real breaking changes or significant new features being introduced in 1.3, changes in the directory structure and code generators introduce a seismic shift in how a developer approaches a Phoenix project. [Chris McCord](http://www.chrismccord.com/) provides an overview and insight into these changes in the [Lonestar Elixir keynote address](https://www.youtube.com/watch?v=tMO28ar0lW8), his [Elixir Forum post](https://elixirforum.com/t/phoenix-v1-3-0-rc-0-released/3947), and his more recent [ElixirConf EU keynote](https://www.youtube.com/watch?v=pfFpIjFOL-I). Those were all essential to getting started, but I found it more easy to understand the advantages once I began implementing it in a project.

While the Phoenix teams does emphasize that these changes are suggestions for how to organize your code, for the purposes of this project I chose to treat it as gospel. For me, these are the three big takeaways from this new version:

1. While Phoenix has only ever been the web layer for your Elixir app, the new directory structure makes that abundantly clear and better defines its role. Everything relating to collecting and serving Phoenix web endpoints (JSON, HTML, websockets, etc.) is neatly packed away within the `lib/my_app` directory, including the `/web` directory.
2. On a similar note, models are gone. Where the web interface was once build around the models, developers now instead create separate interfaces for various aspects of their apps. These context modules exist within their own directories, acting as a boundary for the underlying system of schemas and modules. Your endpoints, located in the `/web` directory, will access these context modules to gather data to serve to users. More on this later.
3. Assets no longer clog up the root directory. All of your external assets are stored in the `/assets` directory. Since I was using Webpack for this project, this was a particular pain for me initially, but ultimately it makes the project significantly easier to navigate. If you’re interested, [here’s how I setup and configured](https://github.com/davelively14/configs/blob/master/phoenix_1.3_react_redux.md) my Phoenix 1.3 app to work with Webpack 2, React, Redux, and React Router.

At its core, Phoenix 1.3 provides a forcing mechanism to design with intent. You have to think about what you’re going to do before you do it. I struggled early with the new structure, but ultimately these guidelines lead to better app development, with clear divisions of responsibility, and more reliable code.

## The stuff dreams are made of

A quick note on the nomenclature I chose. In keeping with the Phoenix mythical bird theme, I named my systems after characters from the classic 1941 film, [The Maltese Falcon](https://en.wikipedia.org/wiki/The_Maltese_Falcon_(1941_film)). The relevant part of the premise is rather simple: A prospective **Client** approaches Sam **Spade** with an interesting case. Spade takes the case and dispatches his partner, Miles **Archer**, to gather information on the quarry, but ultimately it's up to Spade to solve the mystery. Fortunately, we made a few improvements on the movie, so if/when Archer dies, he’ll be seamlessly revived and put back into action thanks to Elixir’s awesome reliability.

![Figure 1](/img/capstone/web_system.png)
<small>**Figure 1.** *`Clients` and `Spade` systems expose their interface to the `Web` system. The `Clients` controllers requests data via the `Flatfoot.Clients` context module (not the `Repo`) before serving via JSON endpoint. Similarly, users may use `SpadeChannel` to request data through the `Flatfoot.Spade` context module. `Spade` `SpadeInspector` and `Archer` .*</small>

In this app’s basic form, the `Clients` system handles all requests relating to the management of users and their preferences, such as creating a profile, editing that profile, and authorizing access via session tokens. The `Spade` system is where the significant action happens. Via websocket connection, users may request retrieval of persisted results or request new results. When those new results are requested, `Spade` will utilize`SpadeInspector` to manage deployment of `Archer` to fetch, parse and return results. Upon receipt of new results, `SpadeInspector` will evaluate those results, store them, and asynchronously return them to the user.

## Organizing in Phoenix 1.3

We can see a basic outline of the architecture quite clearly in the new directory structure. Each system has it's own directory within the `/lib/flatfoot` directory. Also worth noting, the `/web` directory - once a clinger to the root directory in early versions of Phoenix - now finds a more appropriate home.

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

If you’re used to running a third party bundler like [webpack](https://webpack.github.io/) with Phoenix 1.2 and earlier, you’re also likely to notice how uncluttered the root directory is. `package.json`, `webpack.config.js`, and the `node_module` directory are now all packed away quite neatly within the `/assets` directory.

We're left with a clean, easy to reference directory tree, and clearly defined systems.

## Contexts as API Boundaries

As Mikel Myskala has [pointed out](http://michal.muskala.eu/2017/05/16/putting-contexts-in-context.html), the biggest mind shift here is understanding that we're exposing functionality that

![Figure 2](/img/capstone/boundaries.png)
<small>**Figure 2.** *Systems should only be accessed through the functions of their context modules: `Flatfoot.Clients`, `Flatfoot.Spade`, `Flatfoot.SpadeInspector`, and `Flatfoot.Archer`.*</small>

The `Clients` system gives us a better look at we used context modules to manage access across the system boundry. Note that I chose to slightly break from Phoenix 1.3 generator convention by storing all Ecto schemas within the `/clients/schema` directory. While there is no need to do this, I found it easier to read as I added complexity to my systems.

<pre>
|-- clients
|   |-- schema
|   |   |-- blackout_option.ex
|   |   |-- notification_record.ex
|   |   |-- session.ex
|   |   |-- user.ex
|   |-- support
|   |   |-- auth.ex
|   |		
|   |-- <a href="https://github.com/davelively14/flatfoot/blob/master/lib/flatfoot/clients/clients.ex" target="_blank"><b>clients.ex</b></a> <-- click to view in new tab
</pre>

Unlike previous versions of Phoenix, accessing the underlying schema and modules should only be accomplished by calling functions within the `Flatfoot.Clients` module contained within `clients.ex` - sorry, no more `Repo` calls from a controller. Within the context you'll find common, CRUD-like functions like `Clients.delete_user(user)` or `Clients.get_user!(123)`. But you can quickly create more dynamic and useful functions like `Clients.get_user_by_token(token)`, which returns a `User` when provided a valid session token. The `Flatfoot.Clients` context acts as the gatekeeper for the system boundary and it's the only place you should expose the underlying system to the rest of the app.

## Visualizing the Data Flow for a Request

![Figure 3](/img/capstone/data_flow.png)
<small>**Figure 3.** *Data flow overview*</small>

While much of the interaction between the user and backend will be via JSON and websocket to interact with existing data, the purpose of the app is to get information from social media sites, which is what we focus on for this section.

Figure 2 is pretty busy, so we’ll break down this flow of data step by step.

Step 1. An authenticated user will push a request to the SpadeChannel websocket via a `fetch_new_ward_results` message with a `{“id”: ward_id}` payload. `Wards` are individuals the user wishes to keep tabs on. Each `Ward` will have none, one, or many `WardAccounts`, which are individual accounts for various networks (i.e. handles on Twitter, usernames for Facebook).



Step 2. Upon authentication and parameter check, `SpadeChannel` will send send a request to the `SpadeInspector.Server` via the `SpadeInspector.Server.fetch_update(ward_id)` function call. The server starts on app launch, so it’s already running. The server will handle the call internally and retrieve the correct `Ward`. For each `WardAccount` of that `Ward`, the server will build an `mfa` map with instructions for processing and then assign all of the `mfa` maps to a list named `configs`.



Step 3. `SpadeInspector.Server` sends the list of `mfa` maps to the `Archer` system via the `Archer.Server.fetch_data(configs)` function call.

### Something

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
