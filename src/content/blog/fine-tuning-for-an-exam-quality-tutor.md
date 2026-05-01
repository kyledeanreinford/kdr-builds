/---  
title: 'Fine-Tuning for an Exam Quality Tutor'  
description: 'I needed to study for an exam. I ended up fine-tuning a 27B model instead.'  
pubDate: 2026-04-30  
tags: ['ai', 'fine-tuning', 'claude certification', 'showcase', 'llm']  
draft: false
\---

Recently I realized that I would like a better way to study for the GCP Cloud Architect Exam. There are some tools
online and some paid courses, but nothing fits the way I work. I want feedback on why I'm wrong, I want sessions that
track my blind spots and I want to do it for free.

At first I was doing a Claude Project with instructions around how to tutor me. One thing I really liked was that I
could enter my train of thought - show your work, essentially. Then it's not just a correct/incorrect paradigm, it's a "
you were close, but you missed it for this reason" type interaction - much better for learning and retention.

My first attempt was a CLI tutor using docs in ChromaDB and a local LLM generating questions on demand. It was a bit too
slow on my laptop and the results weren't particularly great either.

I had to experiment with what the best models would be for this. All the domain knowledge it needs is right there in the
docs - it doesn't need to know about Rayleigh scattering or who won the World Series in 1959. A smaller model could be
OK here. I started with Gemma4, but that was laggy. Then switched to the Qwen family - Qwen3 4B. Still pretty slow on
the question generation.

---

I ended up pointing things at OpenAI. My hardware at home is just too slow to be able to run something like this
quickly. I also considered using OpenAI to do the initial question creation, and then move over to a local model, but
that's a lot of added complexity and large gap in question generation quality.

This is a tradeoff - I could spend money (big frontend investment) to get better hardware, I could spend time by waiting
for a too-slow model or I could spend a marginal amount of my company's cash to ping gpt-5.4-mini and depend on their
models, which are way better than mine, and way faster.

---

So I decided to shift! It turns out our company needs people to pass the Claude Certified Architect exam to become
Anthropic Partners, so I'm getting rid of all my GCP stuff. Well, actually, I'm going to use GCP question/answer pairs
from online sources and use those to train a model for exam level question creation and use my company's Mac Studio -
I've never fine-tuned before, so this seems like a fun time to try!

---

## Fine-Tuning for Exam Questions

There are a lot of question sets out there, but not many that are good. There are the sample questions from places like
Google's docs and Whizlabs, etc, but I find that they really lack depth when you get to the actual exam. There are also
places that host dumps of questions from exams. Apart from the ethical questions around using that kind of set, there's
the fact that these certification companies change their questions and things get out of date.

That said, they are as close to the actual wording as you're gonna get! And there are a lack of existing CCA Exam
questions because it's so new. So I decided to grab some of those sets - ~150 questions from several different exams -
GCP Cloud Architect, GCP Cloud Developer, Microsoft AI-102 and a couple others. I just took the questions, not the
answers, then ran them through Opus (Claude called me out for using OpenAI for a Claude exam...) to make a base question
set, give me the answer, call out any notes and flag any possible issues, like missing diagrams or case studies that
were not included. When there were issues, I looked at that specific question and figured out what was wrong or right
with it and made sure the answer was correct.

So now I have a collection of about 1000 questions that I can use to train on!

The next question is where do I train? I can do it through OpenAI, but that costs money and is honestly a little less
fun. But while I was out in the Initial Capacity office last month, one of my bosses, Tyson, and I set up a Mac M2
Studio with 64 gigs of RAM to use as a SSH-tunneled AI resource. It's not super fast, but we can run a big Qwen model
and Gemma4 for experimentation in a sandbox that never touches the public internet. For now, it's my best shot at
fine-tuning a local model to do the question generation for me.

---

## Picking The Stack

So for this I have the Mac Studio from afar via Cloudflare Tunnel. Since I'm fine-tuning on a Mac, I can use the Apple's
MLX framework - MLX-LM is essentially their version of PyTorch + Hugging Face. There's a lora method on mlx_lm for
fine-tuning, which seemed like what I needed.

Now I need to pick a model. This isn't a super low-compute task I'm asking the model to do. I'm trying to teach it a
behavior that takes some real thought. Are the questions well structured? Are the distractors plausible? Do they follow
the pattern of using actual product names in the questions as 100% of the certification exams I've taken tend to do?
Then I'm asking it to create reasoning for why each answer is right or wrong. So this can't just be a phi-mini or
something like that. Maybe Llama or Qwen - and something with at least 8 billion parameters. I'm a Qwen fanboy for the
most part and I think Qwen3.5 is one of the most impressive smaller models I've seen, but the architecture its using is
a newer and fine-tuning it is not as well documented. Tuning Qwen3 8B seems like the best first iteration on this data.
 
---

## First Training Run

So after getting my questions annotated and into a train.json file, I was ready to run mlx_lm.lora. With full knowledge
that my M5 Air would fail at the task, I used it just to verify the config. Man did my machine get hot quick! So it was
time to move on to the M2 Studio. After moving the repo over and getting the dependencies installed, I did a first run!
But I forgot that doing a long running process like this over a Cloudflare Tunnel would likely result in a broken pipe
and a stopped process. So I went back with tmux and ran again. Here's how it looked:

```
mlx_lm.lora -c lora_config.yaml
Loading configuration file lora_config.yaml
...
Loading datasets
Trainable parameters: 0.237% (19.399M/8190.735M)
Starting training..., iters: 1200
Calculating loss...: 100%|██████████| 25/25 [01:56<00:00,  4.68s/it]
Iter 1: Val loss 2.133, Val took 116.913s
Iter 10: Train loss 2.003, Learning Rate 1.000e-05, It/sec 0.120, Tokens/sec 115.811, Trained Tokens 9640, Peak mem 14.402 GB
Iter 20: Train loss 1.431, Learning Rate 1.000e-05, It/sec 0.127, Tokens/sec 124.415, Trained Tokens 19463, Peak mem 14.605 GB
...
Calculating loss...: 100%|██████████| 25/25 [01:56<00:00,  4.64s/it]
Iter 100: Val loss 1.049, Val took 116.089s
Iter 100: Train loss 0.999, Learning Rate 1.000e-05, It/sec 0.136, Tokens/sec 125.762, Trained Tokens 95647, Peak mem 14.808 GB
Iter 100: Saved adapter weights to adapters/adapters.safetensors and adapters/0000100_adapters.safetensors.
```

So what's going on there? We're using mlx_lm.lora, a built-in method from Apple's MLX-LM library. Passing in a
lora_config where we define the model to train, the learning rate, number of iterations, etc. After downloading the
model (Qwen3 8b), it lets us know that we can train 0.237% of the parameters. Tiny!

Then we start training. Above you can see the first 100 iterations, out of 1200 total. Every 100 iterations we do
calculations to see how it's going. One thing we can see from the iteration report lines is the Peak mem (peak memory
used) is 14.8 gigs - the m2 isn't even breaking a sweat. Cool! So looking at this data, we want the val(idation) loss
and train(ing) loss both to drop together. Once they diverge, it's time to stop. At that point, the model will just
memorize, rather than continue learning and understanding. The lora method saves checkpoints along the way - every 100
iterations - so we can train past the point of use and use the adaptor that's saved at, say, 700 iterations, if it's the
best one.

In fact, that's what happened. We kept training to 1000 iterations, but the numbers were clear - the training and
validation loss were getting further and further apart. No bueno. So we grab the best one - which was 700 iterations -
and fuse it to the model. `mlx_lm.fuse`. It's quick - just saving the 20mb or so of weights.

> **Sidenote:** I didn't save my training logs the first few times around. I thought it would be easy enough to see them in tmux, but I'd suggest that you `tee` your lora logs to an output file, use the `--report-path` option, or just pipe through something that captures the iteration lines.

So... now I have a fine-tuned model! It exists on the m2 in a models folder and I can try it out.

Let's test it with:

```
mlx_lm.generate \
  --model ./models/cert-tutor-qwen3-8b \
  --prompt "Generate a multiple choice exam question about Claude's tool use capabilities"
```

<details>
<summary>Chain of thought (click to expand)</summary>

```
<think>

Okay, the user wants a multiple-choice exam question about Claude's tool use capabilities. Let me start by recalling what I know about Claude's features. Claude is a large language model developed by Anthropic, right? It can use various tools, but I need to be specific about the types of tools.

First, I should think about the different categories of tools. There are APIs, web search, code execution, and maybe others like data analysis or integration with external services. The question should ask which of these is NOT a tool use capability of Claude. That way, the options can include both correct and incorrect examples.

Wait, the user might be looking for a question that tests understanding of what Claude can and cannot do. So the correct answer would be something that's not a valid tool. For example, Claude can use APIs, perform web searches, execute code, but maybe not something like real-time video processing or hardware control. Let me verify that.

I remember that Claude can access external APIs, perform web searches, run code in environments like Python, and use tools for data analysis. But real-time video processing or controlling physical hardware isn't something it can do. So the incorrect option would be something like that.

Now, structuring the question. The stem should state that Claude can use various tools and ask which is NOT a capability. The options need to include three correct ones and one incorrect. Let me list possible options:

A) Accessing external APIs
B) Performing web searches
C) Executing code in Python environments
D) Controlling physical hardware in real-time

The correct answer would be D. That makes sense because while APIs, web searches, and code execution are valid, controlling physical hardware in real-time isn't a tool use capability of Claude. I should make sure the options are clear and the distractors are plausible but incorrect.

Also, the question should be phrased to avoid ambiguity. Maybe start with "Which of the following is NOT a tool use capability of Claude?" Then list the options. The explanation should clarify why D is incorrect, highlighting that real-time hardware control isn't supported through its toolset.
</think>
```

</details>

```
**Question:**
Which of the following is NOT a tool use capability of Claude?

A) Accessing external APIs to retrieve real-time data
B) Performing web searches to gather current information
C) Executing code in Python environments for computational tasks
D) Controlling physical hardware (e.g., robots, IoT devices) in real-time

**Correct Answer:** D

**Explanation:**
Claude can use APIs, perform web searches, and execute code in supported environments, but it does not have direct capabilities to control physical hardware in real-time. Tool use in Claude is limited to software-based interactions and external data retrieval, not hardware manipulation.
==========
Prompt: 20 tokens, 97.038 tokens-per-sec
Generation: 570 tokens, 69.814 tokens-per-sec
Peak memory: 8.853 GB
```

Fine. Not amazing. But to be fair, it wasn't trained on any Claude stuff - it was trained on exam questions for GCP,
AWS, etc. Lets give it another go but aim it more towards its training content:

```
mlx_lm.generate \
  --model ./models/cert-tutor-qwen3-8b \
  --prompt "Generate a multiple choice exam question about designing highly available cloud architectures"
```

<details>
<summary>Chain of thought (click to expand)</summary>

```
<think>

Okay, the user wants a multiple-choice exam question about designing highly available cloud architectures. Let me start by recalling the key concepts in high availability. High availability in cloud computing typically involves redundancy, fault tolerance, load balancing, and proper region distribution.

First, I need to identify the main components or strategies involved. Common elements include using multiple availability zones, load balancers, auto-scaling, and health checks. The question should test understanding of these concepts.

I should structure the question to ask about a specific scenario or best practice. For example, a common mistake is not distributing resources across multiple zones, so the question could present a scenario where a company is setting up a web application and asks which approach ensures high availability.

Possible options need to include the correct answer and plausible distractors. The correct answer would involve multiple availability zones and a load balancer. Distractors might include single-zone deployments, using only a single load balancer without redundancy, or not using any redundancy at all.

I should also consider the AWS services, since many cloud architecture questions reference AWS. Mentioning services like Elastic Load Balancer (ELB) and multiple AZs would be relevant. However, the question shouldn't be tied to a specific provider, so maybe keep it general but use common terms.

Another angle is to ask about the primary method for ensuring high availability. The answer would be distributing resources across multiple AZs and using a load balancer to route traffic. Other options could be using a single AZ, relying on auto-scaling without redundancy, or not implementing any redundancy.

I need to make sure the question is clear and the options are distinct. Let me draft the question and options. The correct answer should be the one that combines multiple AZs and a load balancer. The distractors should each have a single flaw—like only one AZ, no load balancer, or only auto-scaling without redundancy.

Wait, auto-scaling alone doesn't ensure high availability; it's about handling traffic spikes. High availability is more about fault tolerance and redundancy. So the correct answer should focus on AZ distribution and load balancing. That makes sense.

Let me check if there are other aspects. Maybe cross-region replication or multi-region setups, but the question is about designing highly available architectures, which typically refers to within a single region's AZs. So sticking to multiple AZs and load balancer is appropriate.

I think that's a solid approach. Now, phrasing the question clearly and ensuring the options are well-constructed without overlapping correct elements. The correct answer should be the combination of multiple AZs and a load balancer. The other options each miss one of these components.
</think>
```

</details>

```
**Question:**
Which of the following is the most effective strategy for designing a highly available cloud architecture for a web application?

A) Deploy all servers in a single Availability Zone (AZ) and use a single Elastic Load Balancer (ELB) to distribute traffic.
B) Distribute servers across multiple Availability Zones (AZs) and use a load balancer with health checks to route traffic to healthy instances.
C) Use a single Availability Zone with auto-scaling enabled to handle traffic spikes, relying on the cloud provider's SLA for uptime guarantees.
D) Place all resources in a single region without redundancy, but implement a database replication strategy to ensure data availability.

**Correct Answer:** B
**Explanation:** High availability requires redundancy across multiple Availability Zones (AZs) to protect against single-point failures, combined with a load balancer that performs health checks and automatically routes traffic to healthy instances. Options A and C rely on a single AZ, which fails the high availability requirement. Option D lacks redundancy across AZs, making it vulnerable to regional outages.
==========
Prompt: 20 tokens, 96.866 tokens-per-sec
Generation: 761 tokens, 69.304 tokens-per-sec
Peak memory: 8.902 GB
```

This is quite a bit better. Still a bit generic, but you can see the question style is better. It's not using any RAG
docs, so that could be part of the issue. But now I'm sort of realizing that **I didn't actually train the model on how
to generate a question - I trained it to answer it**. My data was Question -> Answer pairs annotated with reasoning, so
the model learned how to answer and explain, not to generate new ones!

```
mlx_lm.generate \
    --model ./models/cert-tutor-qwen3-8b \
    --prompt "Question: A company runs a web application on EC2 instances behind an ALB. They want to eliminate single points of failure across two availability zones. Which approach should they use?

  A. Deploy all instances in a single AZ with an ALB
  B. Use Auto Scaling across two AZs with health checks
  C. Use a single large EC2 instance with EBS snapshots
  D. Deploy to two regions with Route 53 latency routing

  What is the correct answer? Explain each choice."
```

```
==========
<think>

</think>

The correct answer is B.

A. Deploying all instances in a single AZ with an ALB means the entire application is vulnerable to a single AZ failure. If that AZ goes down, the entire application is unavailable, which does not eliminate single points of failure.
B (correct). Using Auto Scaling across two AZs with health checks is correct because Auto Scaling automatically distributes instances across multiple AZs, and health checks ensure that traffic is routed only to healthy instances. This eliminates single points of failure by having redundant instances in different AZs.
C. Using a single large EC2 instance with EBS snapshots is a single point of failure because if the instance fails, the entire application is down. While EBS snapshots provide some data redundancy, the instance itself is still a single point of failure.
D. Deploying to two regions with Route 53 latency routing provides multi-region redundancy, which is overkill for eliminating single points of failure within a single region. It also adds unnecessary cost and complexity compared to a multi-AZ approach.
==========
Prompt: 114 tokens, 327.484 tokens-per-sec
Generation: 213 tokens, 70.661 tokens-per-sec
Peak memory: 8.883 GB
```

Now if I run that against the base model, here's what I get:

```
mlx_lm.generate \
    --model mlx-community/Qwen3-8B-8bit \
    --prompt "Question: A company runs a web application on EC2 instances behind an ALB. They want to eliminate single points of failure across two availability zones. Which approach should they use?

  A. Deploy all instances in a single AZ with an ALB
  B. Use Auto Scaling across two AZs with health checks
  C. Use a single large EC2 instance with EBS snapshots
  D. Deploy to two regions with Route 53 latency routing

  What is the correct answer? Explain each choice."
```

<details>
<summary>Chain of thought (click to expand)</summary>

```
<think>
Okay, let's tackle this question. The company has a web application running on EC2 instances behind an ALB, and they want to eliminate single points of failure across two availability zones. The options are A to D. I need to figure out which one is correct.

First, let me recall what each option entails.

Option A says to deploy all instances in a single AZ with an ALB. Wait, but if they're all in one AZ, then if that AZ has an outage, the entire application would be down. Also, the ALB is in the same AZ, right? So even if they have multiple instances in one AZ, they're still dependent on that single AZ. That doesn't eliminate single points of failure across two AZs. So A is probably not the right answer.

Option B suggests using Auto Scaling across two AZs with health checks. Auto Scaling across two AZs would mean that instances are spread out, so if one AZ goes down, the other can take over. Health checks would ensure that unhealthy instances are replaced. That sounds like it would eliminate single points of failure because the instances are in different AZs. But wait, the question mentions eliminating single points of failure across two AZs. So maybe this is the correct answer. But I need to check the other options too.

Option C is using a single large EC2 instance with EBS snapshots. A single instance is a single point of failure. Even if you have EBS snapshots, if the instance fails, you can't recover unless you have a backup. But the question is about eliminating single points of failure across two AZs. So this approach is worse because it's a single instance. So C is definitely not the answer.

Option D is deploying to two regions with Route 53 latency routing. That would provide redundancy across regions. But the question mentions two availability zones, not two regions. Also, latency routing is about routing traffic based on latency, but if the primary region fails, you might not have a failover. However, the question is about two AZs, not two regions. So maybe D is not the right answer. But wait, the question says "across two availability zones," so maybe they want redundancy within the same region across two AZs. So D is about two regions, which is more than two AZs. So maybe D is not the correct answer here.

So going back, the correct answer is B. Because using Auto Scaling across two AZs with health checks would ensure that instances are spread across two AZs, and if one AZ has a failure, the instances in the other AZ can take over. The ALB is in the same VPC, so it can route traffic to instances in different AZs. Health checks would ensure that the instances are healthy, and Auto Scaling would replace unhealthy ones. This setup would eliminate single points of failure across two AZs.

Option A is wrong because it's all in one AZ. Option C is a single instance. Option D is about two regions, which is more than the question's requirement of two AZs. So B is the correct answer.
</think>
```

</details>

```
The correct answer is **B. Use Auto Scaling across two AZs with health checks**. Here's the breakdown of each option:

---

### **Option A: Deploy all instances in a single AZ with an ALB**
- **Why it's incorrect**:
  - This approach places all EC2 instances in a single Availability Zone (AZ). If that AZ experiences an outage (e.g., a power failure or network issue), the entire application becomes unavailable.
  - While the Application Load Balancer (ALB) distributes traffic across instances in the same AZ, it does not provide redundancy across AZs.
  - **Single point of failure**: The entire system depends on one AZ, violating the requirement to eliminate single points of failure across **two AZs**.

### **Option B: Use Auto Scaling across two AZs with health checks**
- **Why it's correct**:
  - **Auto Scaling** ensures that EC2 instances are distributed across **two AZs**, providing redundancy. If one AZ fails, the Auto Scaling group automatically replaces unhealthy or failed instances in the other AZ.
  - **Health checks** monitor the instances and ensure that only healthy instances receive traffic. This eliminates single points of failure by spreading the workload and ensuring fault tolerance.
  - The ALB routes traffic to healthy instances across both AZs, maintaining high availability.
  - This setup aligns with the goal of eliminating single points of failure **across two AZs**.

### **Option C: Use a single large EC2 instance with EBS snapshots**
- **Why it's incorrect**:
  - A **single EC2 instance** is a **single point of failure**. Even with EBS snapshots, the instance itself is vulnerable to outages.
  - EBS snapshots provide data redundancy but not application-level redundancy.
  - This approach does not address the requirement for **two AZs** or eliminate single points of failure.

### **Option D: Deploy to two regions with Route 53 latency routing**
- **Why it's incorrect**:
  - While this provides **cross-region redundancy**, the question specifically asks for **two AZs** within a single region, not two regions.
  - **Route 53 latency routing** routes traffic based on latency, but it does not inherently provide failover between regions. If the primary region fails, the secondary region may not automatically take over unless configured with failover routing (e.g., weighted or latency-based with health checks).
  - This approach is more complex and overkill for the requirement of **two AZs**.

### **Conclusion**
**Option B** is the correct answer because it ensures **high availability across two AZs** through Auto Scaling and health checks, eliminating single points of failure. The other options either fail to provide redundancy across AZs or introduce unnecessary complexity.
==========
Prompt: 114 tokens, 333.139 tokens-per-sec
Generation: 1234 tokens, 67.730 tokens-per-sec
Peak memory: 9.012 GB
```

OK! So we can see that our model was trained! It gave us back the exact format we expected and nothing more - 213
tokens. Concise, readable, helpful for someone that would be trying to learn (briefly) why their answer is right or
wrong. In contrast, the untrained model came up with 1234 tokens! It's got a lot of great information, but if I had to
read that while essentially going through flashcards, I wouldn't get very far.

So I trained a model successfully, but I didn't quite train it the way I wanted. I need Claude to restructure my data to
pull out the topic for each question, then have it generate the question, answer and explanations as part of the
fine-tuning.

---

## V2 Training

Alright, here we go with training round 2! I assumed round 2 would start with a higher val loss because generating
questions is further from what an LLM does natually. I checked the numbers. Wrong. Round 1 was 2.133, round 2 was 1.933.
Numbers matter, Kyle.

And we're done! Looks like 700 was the right iteration length again, though I might try again with a higher learning
rate to see what that does.

Here's the first result!

```
mlx_lm.generate \
  --model ./models/cert-tutor-qwen3-8b-v2 \
  --prompt "Generate a multiple choice exam question about Claude's tool use capabilities"
```

```
==========
<think>

</think>

Which of the following is true about Claude?

A. Claude can use tools to perform tasks that require internet access.
B. Claude can use tools to perform tasks that require internet access.
C. Claude can use tools to perform tasks that require internet access.
D. Claude can use tools to perform tasks that require internet access.

Correct Answer: A

Explanation:
A. Correct. Claude can use tools to perform tasks that require internet access, such as looking up information, accessing APIs, or interacting with external services.
B. Incorrect. While this statement is factually true, it is a duplicate of option A and is not a valid multiple-choice question because it repeats the same answer.
C. Incorrect. While this statement is factually true, it is a duplicate of option A and is not a valid multiple-choice question because it repeats the same answer.
D. Incorrect. While this statement is factually true, it is a duplicate of option A and is not a valid multiple-choice question because it repeats the same answer.
==========
Prompt: 20 tokens, 96.820 tokens-per-sec
Generation: 210 tokens, 71.187 tokens-per-sec
Peak memory: 8.765 GB
```

LOL. Big fail. What was the problem? Well, depending on when Qwen3's training date cutoff is, it may not know about
Claude's tool use capabilities. We're not feeding in the Anthropic docs, so the model is confused. I think that makes
sense. Let's do another with a topic the model knows more about:

```
mlx_lm.generate \
  --model ./models/cert-tutor-qwen3-8b-v2 \
  --prompt "Generate an exam-quality multiple choice question about implementing rate limiting for API endpoints"
```

```
==========
<think>

</think>

You are building a RESTful API that will be used by a large number of clients. You want to limit the number of requests that a client can make to your API within a given time period. What should you do?

A. Implement rate limiting on the client side.
B. Implement rate limiting on the server side.
C. Implement rate limiting on the database side.
D. Implement rate limiting on the load balancer side.

Correct Answer: B

Explanation:
A. Client-side rate limiting is unreliable because clients can be spoofed, modified, or bypassed entirely. It does not provide a consistent, enforceable limit across all clients.
B. Server-side rate limiting is correct because it is implemented on the API server itself, ensuring that all clients — regardless of their own configurations — are subject to the same request limits. This is the standard and most reliable approach for enforcing rate limits in a RESTful API.
C. Database-side rate limiting is possible but is not the standard approach. It introduces significant database load and latency, and it is more complex to implement and maintain compared to server-side rate limiting.
D. While load balancers can perform rate limiting, this is not the standard or recommended approach for implementing rate limits on a RESTful API. Load balancers are typically used for distribution and basic traffic control, not for enforcing application-level rate limits.
==========
Prompt: 22 tokens, 105.227 tokens-per-sec
Generation: 279 tokens, 70.857 tokens-per-sec
Peak memory: 8.803 GB
```

And now against the untrained model with the same prompt:

<details>
<summary>Chain of thought (click to expand)</summary>

```
==========
<think>
Okay, the user wants me to generate an exam-quality multiple-choice question about implementing rate limiting for API endpoints. Let me start by recalling what rate limiting is. It's a technique to control the number of requests a user can make to an API within a specific time frame. The goal is to prevent abuse, ensure fair usage, and protect the system from being overwhelmed.

First, I need to think about the key aspects of rate limiting. Common methods include using tokens, sliding windows, fixed windows, and maybe even IP-based or user-based limits. The question should test understanding of these concepts. Also, possible implementation strategies like middleware in web frameworks, using libraries, or database storage.

Now, the question should be clear and have four options. Let me consider the most common approaches. For example, a sliding window algorithm is more accurate than a fixed window because it considers the time frame more precisely. But maybe the question can focus on a specific scenario. Let me think of a scenario where a developer is implementing rate limiting and needs to choose the correct method.

Wait, the user might be looking for a question that tests knowledge of different rate limiting techniques. Let me structure the question around a scenario. For example, a developer is setting up rate limiting for an API and needs to choose the best approach. The options could include different methods like token bucket, fixed window, sliding window, or something else.

Alternatively, the question could be about the correct way to implement it in a specific framework. For instance, in Express.js, using middleware like express-rate-limit. But maybe that's too specific. The question should be general enough for an exam but still test understanding.

Another angle: the question could be about the purpose of rate limiting. But that's more basic. The user probably wants a question that requires applying knowledge. Let me think of a scenario where a developer is choosing between different rate limiting strategies. For example, which method is better for handling bursts of traffic?

Wait, the sliding window algorithm is better for handling bursts compared to fixed window. So a question could be about which method is better for allowing a certain number of requests per minute, considering that some requests might come in a burst. The options would be fixed window, sliding window, token bucket, or something else.

Alternatively, the question could be about the correct way to store rate limit information. For example, using in-memory storage vs. a database. But that's another aspect. Let me check the user's initial request again. They want an exam-quality question, so it should be clear, have four options, and one correct answer with explanations.

Let me draft the question. Let's say the question is about choosing the correct rate limiting strategy. The scenario: A developer is implementing rate limiting for an API to prevent abuse. Which method is best suited for allowing a certain number of requests per minute, while also handling bursts of traffic effectively?

Options:
A) Fixed window with a 1-minute time frame
B) Sliding window with a 1-minute time frame
C) Token bucket with a 1-minute refill rate
D) All of the above

Wait, but the answer would be B, sliding window, because it's better for handling bursts. But I need to make sure the options are accurate. Alternatively, maybe the question is about the correct implementation in a specific framework. For example, in Express, using middleware. But the user might not be specific about the framework.

Alternatively, the question could be about the correct way to track rate limits. For example, using a database or in-memory cache. But again, the user might want a more technical question.

Another approach: the question could be about the correct HTTP status code to return when a rate limit is exceeded. The answer would be 429 Too Many Requests. But that's a common question. However, the user might want something more about implementation.

Wait, the user said "implementing rate limiting for API endpoints," so the question should be about the implementation aspects. Let me think of a scenario where a developer is choosing between different approaches. For example, which of the following is a common method for implementing rate limiting in a web application?

Options:
A) Using a database to track request counts per user
B) Using a token bucket algorithm with a fixed capacity
C) Implementing a sliding window algorithm with a fixed time frame
D) All of the above

But the answer would be D, as all are valid methods. However, the question might need to be more specific. Alternatively, the question could be about the correct approach to handle rate limiting in a distributed system, which would involve using a shared storage like Redis.

Alternatively, the question could be about the correct approach to avoid overcounting in a fixed window. For example, a fixed window might count all requests in a 1-minute window, but if the window is reset, it could lead to overcounting. The sliding window avoids that by considering the time between requests.

So, the question could be: Which rate limiting technique is most effective in preventing overcounting of requests when the same user makes multiple requests at the end of a time window?

Options:
A) Fixed window
B) Sliding window
C) Token bucket
D) Leaky bucket

The answer would be B, sliding window. That's a good question. Let me check if that's accurate. Yes, fixed window can have overcounting if the window is reset, but sliding window uses a time-based sliding window, so it's more accurate.

Alternatively, the question could be about the correct HTTP status code. But I think the sliding window question is better. Let me structure that into a proper exam question with four options and a clear correct answer. Also, include explanations for why the other options are incorrect.

Another possibility: the question could be about the correct way to implement rate limiting in a microservices architecture, which would involve using a centralized rate limiting service. But that's more advanced.

I think the sliding window question is solid. Let me write that out. The question would be: Which rate limiting technique is most effective in preventing overcounting of requests when the same user makes multiple requests at the end of a time window?

Options:
A) Fixed window
B) Sliding window
C) Token bucket
D) Leaky bucket

Correct answer: B) Sliding window

Explanation: The sliding window algorithm tracks requests over a time period, allowing for more accurate rate limiting by considering the time between requests rather than a fixed window. Fixed window can lead to overcounting if the window is reset, while token bucket and leaky bucket are different algorithms that manage tokens or water levels, not time-based windows.

Wait, but the token bucket and leaky bucket are different from fixed and sliding windows. So the options should be accurate. The question is about the technique that prevents overcounting. So the answer is B. That's a good question. Let me make sure the options are correctly labeled and the answer is clear.
</think>
```

</details>

```
**Question:**
Which rate limiting technique is most effective in preventing overcounting of requests when the same user makes multiple requests at the end of a time window?

**Options:**
A) Fixed window
B) Sliding window
C) Token bucket
D) Leaky bucket

**Correct Answer:** B) Sliding window

**Explanation:**
The sliding window algorithm tracks requests over a dynamic time period, adjusting the window to account for the time between requests. This prevents overcounting by ensuring that the rate limit is applied based on the actual time elapsed, rather than a fixed interval. In contrast, a fixed window may reset the count at specific intervals, leading to potential overcounting if requests are concentrated near the end of the window. Token bucket and leaky bucket algorithms manage request rates using token replenishment mechanisms, which are not directly related to time-based windowing.

**Why Other Options Are Incorrect:**
- **A) Fixed window:** May overcount requests if the window resets, as it uses a rigid time frame (e.g., 1 minute) without considering the distribution of requests.
- **C) Token bucket:** Focuses on managing a fixed capacity of tokens, not time-based windowing, and is less effective for precise time-based rate limiting.
- **D) Leaky bucket:** Operates on a fixed rate of processing requests, not a sliding or fixed time window, and does not address overcounting in this context.

This question tests understanding of rate limiting strategies and their trade-offs in handling request distribution over time.
==========
Prompt: 22 tokens, 106.146 tokens-per-sec
Generation: 1753 tokens, 67.427 tokens-per-sec
Peak memory: 9.062 GB
```

Much better! But probably need to refine the prompt and then get the domain docs ingested and used in a RAG context.

Question generation didn't go super smoothly. I was getting a bunch of dupes, questions with similar distractors and the
question quality was not consistent or particularly good. I tried tweaking the prompt, passing in docs one at a time,
scaling up to fine-tuned versions of Qwen3 14B and Qwen3 32B. I added a Gemma4 pruning step. There were some
improvements as the models grew, but nothing really got me to where I wanted to be. After all that I was ending up with
only about 60 questions. They were decent, but they didn't cover all the domains.

---

## Splitting The Project In Two

Something had been niggling in the back of my head during this whole process, and I finally realized what it was: this
is actually two separate projects - two different tools. One is the generation - that project could get fed docs and
create a robust question bank, do some passes to make sure all the best questions are there, then save them as a json
payload. Then the second project is the study app. It doesn't need any LLMs. It doesn't even need the internet. It just
needs a json payload and could be done all locally. That's cool!

---

## The Breakthrough

OK I fine-tuned Qwen3.6 - the new (at the time at least!) 27B dense model. This is the model I ended up using for the
final question generation.

The first training run looked good at first, but after a few hundred iterations, things went south. It turns out that
the learning rate of 5e-6 was too high and the val loss was going up! It's supposed to go down. Womp womp. So I changed
the learning rate to a much smaller 1e-6 and let it run for 2000 iterations. The next morning I woke up and took a look
at the data. The val loss kept going down alongside the training loss, so no overfitting. I may have been able to do
more than 2000, but this seemed good enough to try.

I tried to do the question generation, but it was an immediate fail. Qwen3.6's Thinking Mode threw a wrench in the
plan - rather than returning the questions, I was getting the internal monologue. Adding `enable_thinking=False` fixed
it.

But then I dealt with question quality. There were questions about things like Spring AI and .clinerules. Something
still was off - our strategy of creating questions was causing some hallucinations, it seemed.

Around this time I found the official exam guide on the [Anthropic Skilljar](https://anthropic.skilljar.com/) website
and decided to use that to drive the question generation rather than the vector db of docs I had pulled from the site
previously. This led to a much better outcome! I'm not sure if having too many docs vs a tighter exam guide is what
helped reel it in, but doing it this way led Qwen3.6 to follow directions better and write genuinely believable exam
questions. I'm guessing a big part of that was that the exam guide is more explicit about what's in or out of scope -
the LLM had less opportunity for drift. I could use the weights for how much of the exam was used by each domain and
generate questions in chunks.

Even this had an issue. The model would generate 5 questions based on the same bullet point and I'd just have dupes. So
then I had to randomize which bullet point each generation used, which gave more variety. I also had to explicitly tell
it "do not copy these scenarios".

I used this v2 generation script with both Qwen3.5 32B and Qwen3.6 27B and compared the output. 3.6 gave us less
duplication, better distractors, more concrete scenarios and better explanations.

Here's the same task statement (1.6 — design task decomposition strategies for complex workflows) generated by each
model. Same concept, same prompt structure — different feel.

<details>
<summary>Qwen3.5 32B Fine-Tuned (gen-0068)</summary>

**Scenario:** Research Agent for Product Development. Your team is building a research agent to explore new product
ideas for a consumer electronics company. The agent must gather market trends, analyze competitor products, and generate
a product concept. The agent has access to a knowledge base of past product launches and a set of tools for web search,
sentiment analysis, and competitor pricing. The agent is currently producing product concepts that are either too
generic or too narrow in scope. What should you do?

**Options:**

- A) Use a fixed sequential pipeline that first gathers market trends, then analyzes competitor products, and finally
  generates a product concept based on the fixed order of steps.
- B) Use a dynamic adaptive decomposition approach that first gathers market trends, then conditionally analyzes
  competitor products only if the trends indicate a competitive threat, and then generates a product concept based on
  the findings.
- C) Use a fixed sequential pipeline that first generates a product concept, then gathers market trends, and finally
  analyzes competitor products to validate the concept.
- D) Use a dynamic adaptive decomposition approach that first generates a product concept, then conditionally gathers
  market trends only if the concept is novel, and finally analyzes competitor products to validate the concept.

**Correct answer:** B

**Explanation:**

- A dynamic adaptive decomposition approach (B) is correct because it allows the agent to conditionally analyze
  competitor products only when the market trends indicate a competitive threat, preventing both overly generic and
  overly narrow product concepts by adapting the workflow based on intermediate findings.
- Option A is wrong because a fixed sequential pipeline forces competitor analysis regardless of relevance, leading to
  unnecessary or redundant work that can narrow scope.
- Option C is wrong because generating a product concept first and then gathering market trends is logically inverted
  \u2014 the concept should be informed by trends, not the other way around.
- Option D is wrong because generating a product concept first and then conditionally gathering market trends is also
  inverted, and the conditional logic is based on novelty rather than competitive threat, which doesn't address the root
  cause of scope issues.

</details>

<details>
<summary>Qwen3.6 27B Fine-Tuned (gen-0019)</summary>

**Scenario:** Automated Compliance Audit Agent\nYour team is building an agent to audit vendor contracts for regulatory
compliance. The workflow involves three steps: (1) Extracting key clauses, (2) Checking those clauses against a static
regulatory database, and (3) Generating a final report. You notice that 40% of contracts are immediately disqualified
because they lack a required 'Data Residency' clause. Currently, the agent always performs the full three-step pipeline
for every contract, even when the Data Residency clause is missing, wasting compute on the expensive regulatory database
check. What is the most effective architectural change to optimize this workflow?

**Options:**

- A) Switch from a fixed sequential pipeline to a dynamic adaptive decomposition strategy where the agent evaluates the
  extracted clauses after step 1 and conditionally skips the regulatory database check if the Data Residency clause is
  missing.
- B) Keep the fixed sequential pipeline but add a pre-processing step that uses a separate, smaller model to scan for
  the 'Data Residency' keyword before the main agent begins, routing non-compliant contracts to a fast-reject path.
- C) Implement a parallel processing pipeline where the clause extraction and regulatory database check run
  simultaneously, using a timeout mechanism to cancel the database check if the extraction step fails to find the
  required clause.
- D) Increase the context window and system prompt instructions to tell the agent to 'be efficient' and skip unnecessary
  steps, relying on the LLM's internal reasoning to decide whether to query the regulatory database during the
  extraction phase.

**Correct answer:** A

**Explanation:**

- A is right because it directly addresses the concept of choosing between fixed sequential pipelines and  
  dynamic adaptive decomposition. The scenario describes a workflow where intermediate findings (missing clause)
  should  
  determine subsequent steps, which is the exact use case for dynamic adaptive decomposition. A fixed pipeline (prompt  
  chaining) executes all steps regardless of intermediate results, leading to wasted compute on the expensive database  
  check.
- Option B is wrong because it adds unnecessary complexity (a separate model) when the existing agent can handle  
  the conditional logic; it also doesn't address the core architectural choice between fixed vs. dynamic decomposition  
  within the agent's workflow.
- Option C is wrong because parallel processing doesn't solve the problem of conditional  
  execution based on intermediate findings; it actually risks running the expensive database check concurrently with  
  extraction, potentially wasting resources before the cancellation timeout fires.
- Option D is wrong because relying on  
  vague instructions like 'be efficient' in a fixed pipeline is unreliable; LLMs in fixed prompt-chaining
  architectures  
  execute steps sequentially as defined, and without explicit conditional branching logic (dynamic decomposition),
  they  
  cannot reliably skip downstream steps based on upstream results.

</details>

The 32B question is fine - abstract product strategy, generic distractors that all read similarly, an explanation that
mostly restates the answer. The 27B question lands on a specific failure mode (wasting compute on a 40% reject rate),
gives distractors that each represent a real architectural pattern someone might actually try, and the explanation walks
through *why* each wrong answer is wrong rather than just declaring it. That's the difference 3.6 made across the whole
set.

The next step was a quality gate. Let's prune bad questions out of the set. To do this, we used Gemma4 26B as a judge.
First I thought we should go down to the best 100 questions, but then we ended up with a question set that was very
different from the domain weights in the exam guide. The move was really to not give a limit on the best questions -
just to prune out bad ones using a quality threshold and deduping. Our 205 questions ended up at 191 after this - a nice
full set! The distribution ended up as d1=29% / d2=16% / d3=19% / d4=20% / d5=16%. That's within two percentage points
of the exam weights.

I wanted to do one last pass with a frontier model to make sure I didn't have any bad data in there, so I had Opus take
a look. It pruned out another 7 questions, bringing the working total to 184. Add in the Anthropic example questions and
some community ones and we're just above 200.

So some things I've learned:

* Tweaking the learning rate has serious consequences for overfitting model training
* The exam guide worked much better than the RAG with all of Anthropic's docs
* Bullet point targeting prevents redundancy
* Pruning needs both a quality floor and a dedup pass
* The judge model matters - Opus caught hallucinations a 26B local model couldn't

## The App

So with the finished batch of questions, it was time to implement the study app. I called
it [CCA Tutor](https://github.com/kyledeanreinford/cca_tutor). It's a command line tool using Python's Rich library and
uses the generated questions as a json file - much less overhead than having to talk to an API or wait for a laggy LLM
experience.

Claude built it for me over several iterations. There's a study mode that randomizes the questions, tells you if you're
right or wrong (with reasoning built by the question generation), and keeps track of which domains are weak spots for
you. There's also an exam mode that gives 60 questions in 120 minutes for a bit of added pressure.

## Conclusion

At the beginning of this post I outlined a few things I'd like:  
I want feedback on why I'm wrong, I want sessions that track my blind spots and I want to do it for free.

How did I do?

Sessions that track my blind spots is a win. I feel really good about how that all works. The tutor has a menu bar at
the top that shows percentages for each of the 5 CCA domains. I even have a feature to be able to study one domain
specifically.

Feedback on why I'm wrong isn't quite as clear cut. While I do have reasoning for why each answer is right or wrong, one
thing that was nice about using an LLM in real time is that I could say "I think A is right because of such and such"
and the response I got back would address that. No such possibility here without a direct line to AI, but still much
better than a red x or a green checkmark.

As for doing it for free... I'm not using up Claude tokens every time. I didn't have to buy any additional hardware. I
used a ton of compute at my office and a lot of my own time, but the tradeoff is that I learned a lot.

Did I pass the exam? I haven't taken it yet - I spent all my study time building this tutor... Claude
just [draws you in](https://kyledeanreinford.com/blog/falling-for-claude)...
