# RankedAPI

RankedAPI is an A/B testing tool for SEO.

## Why use RankedAPI?

RankedAPI improves SEO strategies in two ways:

1. Discovering promising SEO opportunities which justify further investment, and
2. Flagging SEO tactics which might actually hurt your overall search rankings.

## Up and running with RankedAPI in five minutes

Let's say we maintain a site called *Phone Photography*, built with a typical web MVC framework. The site has a huge blog to draw in an audience, an email intake form to build an email list, and a paid members-only section with courses. Our goal with *Phone Photography* is to convert the audience to our email list, and then convert our email list subscribers to paid members. Our audience growth strategy is primarily SEO. We want to improve our SEO to increase conversions across the entire funnel.

We want to improve our rankings so we can grow our audience. We have a hypothesis: adding `alt` attributes to `<img>` tags in blog posts will improve rankings.

Let's see how RankedAPI can either confirm or reject the hypothesis.

### Step 1: Authenticate

First, get an API key and secret. Go to the [signup page](#) to request them. Let's say you get back the API key `api_key` and the API secret `api_secret`.

### Step 2: Create a new experiment

Then, create an experiment. You can do this from your command line:

```bash
$> curl https://api.rankedapi.com/v0/experiments \
      --request POST \
      --user api_key:api_secret \
      --data-urlencode "hypothesis='Keywords in image alt attributes improves pagerank'"
      # Notice that each experiment tests a hypothesis
```

An authenticated POST to the `/experiments` endpoint will give a simple JSON blob in the response body:

```json
{
  exp_id: 456
}
```

RankedAPI just created a new A/B testing experiment for you with the ID `456`.

### Step 3: Create the control variation

Now the experiment needs a control. RankedAPI's canonical control variation has the ID `0`. The ID `0` is assigned to the first variation created for every experiment.

You can create the control from your command line:

```bash
$> curl https://api.rankedapi.com/v0/variations \
      --request POST \
      --user api_key:api_secret \
      --data "exp_id=456" \
      --data-urlencode "description='no alt attributes on images'"
      # We added a description for your control variation
```

The response body will contain this JSON:

```json
{
  var_id: 0
}
```

### Step 4: Create a test variation

You can create another variation by sending a second POST request to `/variations`:

```bash
$> curl https://api.rankedapi.com/v0/variations \
      --request POST \
      --user api_key:api_secret \
      --data "exp_id=456" \
      --data-urlencode "description='add alt to images'"
      # Notice the exp_id is unchanged, but we changed the description to match the test variation!
```

We sent another POST request, again to `/variations` with the same `exp_id`, but a new description. The response body's JSON would be:

```json
{
  var_id: 1
}
```

### Step 5: Integrating the experiment

Each blog post in *Phone Photography* is between 500-1000 words and has 2-3 photos. The source code in the controller might make the following objects available to the view:

```python
# Before integrating RankedAPI

blogpost_data["images"] = [{
  src: "/img/breakfast-plate.png"
}, {
  src: "/img/walking-trail-in-forest.png"
}, {
  src: "/img/kids-and-puppies.png"
}]
```

When integrated, RankedAPI will deterministically assign a `var_id` for each page. That is, if you request a `var_id` with the same `exp_id` and `url`, RankedAPI will always return the same `var_id`.

In this example, RankedAPI will ensure that half of *Phone Photography*'s blog posts will be assigned `var_id = 0` to be in the control group. The other half will be assigned `var_id = 1` to be in the test group.

We bake this into the controller:

```python
# After integrating RankedAPI

var_id = RankedAPI.sample({ exp_id: 456, url: blogpost_data["url"] })
alt = "" if var_id == 0 else (" ").join(blogpost_data["keywords"])

blogpost_data["images"] = [{
  src: "/img/breakfast-plate.png",
  alt: alt
}, {
  src: "/img/walking-trail-in-forest.png",
  alt: alt
}, {
  src: "/img/kids-and-puppies.png",
  alt: alt
}]
```

Let's say that in the view source code, rendering each image's `<img>` tag is like this:

```python
def render_image_tag(img):
  # string interpolation in python
  return "<img src='%(src)s' alt='%(alt)s'>" % img
```

So, pages with `var_id = 0` will have empty `alt` values, while pages with `var_id = 1` will have `blogpost_data["keywords"]` as their values.

### Step 6: Monitoring the experiment's results

The experiment is live and underway the moment you deploy your code in Step 5! You can check the results by sending a GET request to the `/results` endpoint:

```bash
$> curl https://api.rankedapi.com/v0/ \
      --request GET
      --user api_key:api_password
      --data "exp_id=456"
```

The response body will be a JSON object. A successful experiment which validates the experiment's hypothesis (and rejects the null hypothesis) will look something like this:

```json
{
  status: "test beat control",
  recommendation: "implement test behavior"
}
```

You can see more information by adding `verbose=true`:

```bash
$> curl https://api.rankedapi.com/v0/ \
      --request GET
      --user api_key:api_password
      --data "exp_id=456"
      --data "verbose=true"
```

This might return:

```json
{
  status: "test beat control",
  recommendation: "implement test behavior",
  variations: {
    "0": {
      average_ranking_before_experiment: 15.1,
      average_ranking_after_experiment: 15.1,
    },
    "1": {
      average_ranking_before_experiment: 15.1,
      average_ranking_after_experiment: 7.4,
    }
  }
}
```

In the JSON return object, if you look at variation 1, you can see that RankedAPI proved the test variation outperformed the control variation. If you look at variation 0, you can see that the control variation's performance was unchanged. Any A/B testing framework should prove its data integrity by showing that the control is unchanged.

Congratulations! You now know with certainty that adding `alt` attributes will improve *Phone Photography*'s ranking!

## Terminology

- Control variation: The pages which will be unchanged to prove whether to validate or reject the experiment's hypothesis
- Experiment: An A/B test with a hypothesis to test, using one control variation and at least one test variation
- Variation: The umbrella term to describe both control and test variations
- Test variation: The pages which will be changed and compared to the control variation to prove whether to validate or reject the experiment's hypothesis
