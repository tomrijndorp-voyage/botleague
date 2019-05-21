# Bot League

## Open league for AI competition

This repo coordinates an open league where bots are ranked on a set of community 
contributed problems. We provide an ongoing leaderboard of the best bots in the 
world on the problems and challenges submitted. Anyone can submit
a problem and and anyone can submit a bot.

### Conceived for self-driving
The league was conceived specifically for self-driving where industry feedback to academia is limited, despite academia being the major contributor of new algorithms to self-driving.
With Bot League, a self-driving car company can present it's toughest problems directly to the public and 
to academia to compete on. 

### Generality
Generality across problems is a key component of the way the league is structured, 
where bots list a set of problems they can solve. Crucially, these problems
can be within completely different simulators on different machines architectures, etc... 
So if  a self-driving car company wants to test a neural net on its ASIC hardware,
it can pull the docker container containing the weights, compile the net
and evaluate it. The platform is agnostic to both bot and problem implementation.
In this way it's more of an API for comparision and competition of AI.
Bots are ranked and displayed on the  
[leaderboards](https://botleague.io) which are [open source and open data](https://github.com/voyage/leaderboard-generator/data).

We provide a reference implementation of bots and problems via Deepdrive, and [TBA].
 
### How it works

High level: Submit a docker container with your bot and problems it solves, 
and get ranked on botleague.io.

Low level:

Pull requests to [bots](bots) in this repo trigger evaluation on the set of [problems](problems) designated in its bot.json. Pull requests are then evaluated, ranked on the leaderboards, and merged so long as they are in the correct format. Bots include a docker container and optionally writeups and source code that are pulled in and tested by problem evaluators. Anyone can contribute a problem, so long as they support the minimal API. Example problem evaluators are currently Deepdrive and [TBA]. To create a problem within one the problem providers, refer to the docs of that provider.

_Side note_: Reproducibility is core to making progress in research, but we put the onus of that on the problem implementations, and choose to avoid enforcing it here for now in the name of simplicity.

## Bots

### Submission

Submit a pull request to this repo with a bot.json under [bots](bots)/{github-name}/{repo-name} - with a [bot.json](crizcraig/forward-bot/bot.json) to have your bot 
evaluated and ranked on the leaderboards.
 
You must push the docker tag referred to in your bot.json before submitting the pull request.
 

## Problems

A list of problems to test your bot against can be found
in [problems](problems).

TODO: Link to problems page on leaderboards site to give an idea of what they involve

A bot-league compatible endpoint mush be included in a new problem.

### Problem Endpoints

Problem endpoints implement a simple API, accepting one request and sending a confirmation then results to liaison.botleague.io. An example problem endpoint can be found at https://github.com/deepdrive/eval.deepdrive.io

#### 1. Accept `/eval` POST

`https://your-endpoint/eval/[problem_name]`, i.e. `/eval/domain_randomization`

with JSON payload containing the following:

`seed`: A random number between 0 and 1 million. This is broadcast to all of 
the problem evaluators for the bot submission.

`eval_key`: Secret problem evaluation key unique to the evaluation of the bot 
on this problem. Used to communicate back to liaison in `/confirm` and `/results` requests.

`docker_tag`: i.e. `your-dockerhub-name/your-bot` or `gcr.io/your-dockerhub-name/your-bot`

`pull_request`: i.e.:
```
  "pull_request": {
    "url": "https://api.github.com/repos/deepdrive/botleague/pulls/4",
    "number": 4,
    "updated_at": "2019-05-21T21:45:51Z",
    "merge_commit_sha": "50f0f2d44e836f30d2815141c866cd2775bb8ec6",
    "head_commit": "4dd65418bef710f5e668c33205040d6f40e7ef43",
    "head_full_name": "deepdrive/botleague",
    "base_commit": "a1e475acfe1218482e27fb013317dd01e0fbfcbf",
    "base_full_name": "deepdrive/botleague"
  }
```

#### 2. Send `/confirm` POST

Problem evaluators must then send a confirmation request with the `eval-key` to `https://liaison.botleague.io/confirm` to verify that botleague indeed initiated the evaluation.

#### 3. Send `results.json` POST

Finally evaluators POST `results.json` to `https://liaison.botleague.io/results` with the `eval-key` to complete the evaluation and to be included on the Bot League leaderboards. An example `results.json` can be found [here](problems/example_results/results.json).

### Problem versioning

In order to support testing new versions of problems (i.e. some new sim commit), Botleague will cancel currently running evaluations against the old version when a change is made to `problem.json` by sending a /cancel_eval request from the  and do a **Problem CI Run** where all ranked bots are retested against the new problem version. Bot submissions against the problem will be marked as pending until the problem CI is complete. If another problem update comes in while in “problem CI mode”, it will wait until the intermediate version finishes.

For purposes of explanation, think of an endpoint in this form

 [https://example.com/{problem_id](https://example.com/{problem_id)}

i.e. [https://example.com/difficult_problem](https://example.com/difficult_problem) - Current commit version `824914daaaee006a99086ff63b2544e3b5f55fab`

Where the problem.json is at

[https://github.com/botleague/boteague/example/difficult_problem/problem.json](https://github.com/botleague/boteague/example/difficult_problem/problem.json)


### Cases for new rank ordering 

#### Rank ordering is the same on new and previous version (common case)

**Resolution:** The new problem version becomes the default version, old scores are stored along with the problem.json commit and new scores are the default scores shown on the leaderboards. Points will be based soley from ranking, not score values, so the users' and bots' points will not change.


#### Rank ordering changes

**Resolution:** Eval/CI fails, problem.json updated with `"archived": true`, and new bot submissions will be disabled for this problem. Problem endpoints can create a new problem if they intended for this change, in which case the old problem will remain archived, or fix the bug and submit a pull request with `"archived": false` in problem.json to trigger another **Problem CI Run**. If the rank ordering changes on a bot pull request, as opposed to a problem pull request, we pause bot submissions on the problem in the same way, and notify the `problem.json` `"contact"` that the problem has been archived until the original ranking has been restored. 

We will avoid requiring score parity for now across problem versions. We will copy bot containers, so will always be able to test the old bots and get new scores.

## Challenges

Challenges are sets of problems a bot is tasked with generalizing across. 

## Testing for determinism

For now we will handle this within the problem endpoint, although we should implement this functionality in a modular way as most problems will want some version of this, likely with different numbers of job runs based on resource constraints and variability of the evaluation.

For example Deepdrive, we kick off **five** simultaneous runs for each evaluation, and use the median score, video, recording for the final score (also returning the scores, videos, and recordings for each of the five).
