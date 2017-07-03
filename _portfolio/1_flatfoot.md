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

_Flatfoot_ operates as a single-page application (SPA) that interacts with the backend via a JSON API for user authentication and profile management, and websocket for all things related to monitoring social media accounts. The basic flow is that a user will create an account and navigate the dashboard. From there, users create a `ward`, which represents a person the user wishes to track. For each `ward`, the user creates `ward_accounts` that represent a social media account for a given backend. The user can then instruct the app to check for new results. Asynchronously, the app will return those results, each with a rating from 0 to 100, where the higher the rating the more potential that the post contains an example of bullying or an ideation.

#### Nomenclature

![Maltese Falcon, 1941](http://www.filminamerica.com/Movies/TheMalteseFalcon/falcon06.jpg)

In keeping with the Phoenix mythical bird theme, I named my systems after characters from the classic 1941 film, [The Maltese Falcon](https://en.wikipedia.org/wiki/The_Maltese_Falcon_(1941_film)). The relevant part of the premise is rather simple: A prospective **Client** approaches Sam **Spade** with an interesting case. Spade takes the case and dispatches his partner, Miles **Archer**, to gather information on the case and report back, but ultimately it's up to Spade to solve the mystery. Fortunately, we made a few improvements on the movie, so if Archer just happens to die, he’ll be seamlessly revived and put back into action thanks to the magic of OTP.

I settled on the name 'Flatfoot' for the app. I realize that's more a beat cop than a private investigator like Spade, but the prevailing slang for a PI at the time doesn't seem appropriate in a modern context.

#### Database Layer

Since this is an app I built in order to further my education, I used a release candidate version of my web framework, Phoenix 1.3.0-rc. I write in more detail about it in a [blog post](http://www.resurgens.io/2017/06/26/phoenix-1-3.html), but the big takeaway here is that models are gone and systems are in. As such, the tables in our database look a little different.

<img align="right" src="/img/flatfoot-2.png">

Instead of a table named `backends`, for instance, we have `archer_backends`. While that indicates that the table is managed by the `Archer` system, other systems can - and do - interact with the tables via their own, system specific schemas.

#### React routing

As a SPA, we only handle routes virtually. In this case, we use the famous [`react-router`](https://github.com/ReactTraining/react-router/tree/master/packages/react-router). In order to ensure our Phoenix server plays well with `react-router`, we had to route any unexpected requests to the root. At the end of the `web\router.ex` file, we placed the following code:

```elixir
scope "/*path", Flatfoot.Web do
  pipe_through :browser # Use the default browser stack

  get "/", PageController, :index
end
```

It's important that block is at the end of our router file, otherwise every request will route to the root. With that in place, we can setup our routes in `lib/flatfoot/web/static/js/react.js`:

```javascript
render(
  <Provider store={store}>
    <Router history={history}>
      <Route path="/" component={App}>
        <IndexRoute component={Landing} />
        <Route path="login" component={Login} />
        <Route path="new-user" component={NewUser} />
        <Route path="profile" component={Profile} />
        <Route path="logout" component={Logout} />
        <Route path="dashboard" component={SpadeChannel} />
      </Route>
    </Router>
  </Provider>,
  document.getElementById('react')
);
```

The first four route paths (`login`, `new-user`, `profile`, `logout`) all deal with the JSON API. The final route, `dashboard`, interacts with our Phoenix Channel via websocket, and that's where all the magic happens. But before we can create magic, we'll need to gain authorization.

#### User management via JSON

All aspects of managing the user profile - creation, editing, preference setting, session management - is handled via Phoenix JSON API. All substantial functionality of the app is only available to authenticated users, via a url safe [base-64 token](https://github.com/patricksrobertson/secure_random.ex) for JSON requests, or a [Phoenix token](https://hexdocs.pm/phoenix/Phoenix.Token.html) for access to the dashboard channel. In order to get the first token, a user must create a profile.

![Figure 1](/img/flatfoot-3.png)
<small>**Figure 1.** *Data flow for creating a new user*</small>

In the example above, a user will make a `new_user` API call, which will be handled by `Web` system's `Router` and routed to the `UserController` module. The `create` function of that module will use the `Clients` context module to create a new user, receive the new user, and then make a subsequent call to `Clients` to login, which creates a base-64 token and returns it within a `%Session{}` structure. The `UserController.create` function then returns a rendering of the session via `Web.SessionView`. The result the user receives looks like this:

  ```json
  {
      "data": {
          "token": "eWE0aEx2eVpGTTBYeHlqWnV1VnZSUT09"
      }
  }
  ```

Our frontend stores that token in the Redux store in `session.token` and uses that in all subsequent calls to our JSON API as part of the headers:

```json
Authorization: Token token="eWE0aEx2eVpGTTBYeHlqWnV1VnZSUT09"
```

To read further about the JSON API functionality, take a look at the [docs here](https://github.com/davelively14/flatfoot/blob/master/JSON_api.md).

Upon receipt of a new token, our frontend will set it within the Redux managed store. Additionally, we use `universal-cookie` to store the token in a cookie. Here's what it looks like in `lib/flatfoot/web/static/js/components/clients/login.js`:

```javascript
import Cookies from 'universal-cookie';
import { fetchUser, fetchPhoenixToken } from './helpers/api';
...
function(text) {
  let token = JSON.parse(text).data.token;
  const cookies = new Cookies();
  cookies.set('token', token, {
    path: '/'
  });

  setToken(token);
  fetchUser(setUser, token);
  fetchPhoenixToken(setPhoenixToken, token);
  browserHistory.push('/dashboard');
}
```

We retrieve from cookies when the app is launched, via `lib/flatfoot/web/static/js/components/app.js`:

```javascript
import Cookies from 'universal-cookie';
import { fetchUser, fetchPhoenixToken } from './helpers/api';
...
const mapStateToProps = function(state) {
  return {
    loggedIn: state.session.token ? true : false
  };
};
...
componentWillMount() {
  const cookies = new Cookies();

  if (!this.props.loggedIn && cookies.get('token')) {
    let token = cookies.get('token');

    this.props.setToken(token);
    fetchUser(this.props.setUser, token);
    fetchPhoenixToken(this.props.setPhoenixToken, token);
  }
}
```

Note how in both files above, we call the `fetchPhoenixToken()` function from our `./helpers/api.js` file, which is required to connected to websocket. More details about how that works can be found in the [Getting Started](https://github.com/davelively14/flatfoot#getting-started) section of the README.

#### SpadeChannel

Our Phoenix Channel, `SpadeChannel`, is the gateway for where all the magic happens. Our React app joins the channel via `lib/flatfoot/web/static/js/components/spade/spade_channel.js`, which is at the end of the `dashboard` route as handled by `react-router`. While the many components of our React app will push requests to the channel, it is within `spade_channel.js` that we listen for all the results.

This will be easier to understand with an example.

![Figure 2](/img/flatfoot-4.png)
<small>**Figure 2.** *Data flow when requesting new results.*</small>

Our `WardDetail` component contains a function that is called when the user clicks a button:
```javascript
fetchNewResults() {
  this.props.channel.push('fetch_new_ward_results', {ward_id: this.props.ward.id});
}
```

When our Phoenix `SpadeChannel` receives the message, it works with the rest of our Elixir app (more on that in the next section) to fetch new results. As those results are received, the send those results to the Phoenix `SpadeChannel`, which then broadcasts to the channel our React app is connected to.

When initially joining, `spade_channel.js` set several event listeners. One of them is set to handle the `new_ward_results`:

```javascript
channel.on('new_ward_results', (_resp) => {
  this.props.clearWardResults();
  channel.push('get_ward_results_for_user', {
    token: this.props.session.token
  });
});
```

A complete guide to working with the channel can be found in the [docs here](https://github.com/davelively14/flatfoot/blob/master/spade_channel.md).

#### Elixir OTP

While React and Phoenix play well together up front, the heavy lifting is done by Elixir.

![Figure 3](/img/flatfoot-5.png)
<small>**Figure 3.** *How Flatfoot gathers data*</small>

We'll pick up at step 2, where our channel calls a function from the `SpadeInspector` context module to pass on notification of the request. The SpadeInspector system contains an Elixir OTP system (based on [Erlang OTP](http://learnyousomeerlang.com/what-is-otp)) consisting of a `Supervisor` and a `SpadeInspector.Server` - a [GenServer](https://hexdocs.pm/elixir/GenServer.html). In this step, `SpadeInspector.Server` receives a request to fetch results for a ward and builds a configuration for each:

```elixir
def handle_cast({:fetch_update, ward_id}, state) do
  configs = if ward = Flatfoot.Spade.get_ward_preload(ward_id) do
    ward.ward_accounts |> Enum.map(fn ward_account ->
      last_msg_id = ward_account.last_msg || ""

      %{mfa: {
          ward_account.backend.module |> String.to_atom,
          :fetch,
          [self(), %{user_id: ward.user_id, ward_account_id: ward_account.id, backend_id: ward_account.backend.id}, ward_account.handle, last_msg_id]
        }
      }
    end)
  end

  if configs, do: Archer.fetch_data(configs)
  {:noreply, state}
end
```

Step 3 begins on the next to last line: `Archer.fetch_data(configs)`. Archer is similar SpadeInspector in that it has an OTP system, but in addition to a server `ArcherSupervisor` is also also supervises a [Task.Supervisor](https://hexdocs.pm/elixir/Task.Supervisor.html) named `FidoSupervisor`.

```elixir
# Children initialized for ArcherSupervisor
children = [
  supervisor(Task.Supervisor, [[name: Flatfoot.Archer.FidoSupervisor]]),
  worker(Flatfoot.Archer.Server, [ self() ])
]
```

`Archer.Server` will tell `FidoSupervisor` to launch each backend according to the provided config (step 4). `FidoSupervisor` will then launch and supervise each backend concurrently (step 5). When each backend retrieves a result (in this case, we only have `Twitter` running), it will parse and send those results back to `SpadeInspector.Server` for processing (step 6).

Upon receiving a result, `SpadeInspector.Server` will add each result, assign a rating, and then broadcast those results to `SpadeChannel` (step 7). `SpadeChannel` will then broadcast those result to our connected frontend (step 8.)

A quick note on scoring. `SpadeInspector.Server` pulls ratings from several csv files containing flagged words and stores them locally via Erlang Term Storage, or [ets tables][http://erlang.org/doc/man/ets.html]. Those `:ets` tables allow for very quick access to a library of over 4,000 negative words that we use to score the rating for the results. You can read more about the speed of ETS' use of Judy Arrays in [this paper](http://erlang.org/workshop/2003/paper/p43-fritchie.pdf).

## Conclusion

_Flatfoot_ provides users with an easy to way to proactively monitor social media accounts for troublesome content without sacrificing anyone's independence. While there is still room for improvement (add support for more social media outlets, tweak the algorithm, add push notification via email or text, etc.), this tool does achieve its main objectives. Additionally, I was able to learn more about a full suite of languages and frameworks, primarily React, Redux, Phoenix, Elixir, and Erlang. This capstone has been a huge learning point for me and I'm grateful I was able to spend so much time developing it.
