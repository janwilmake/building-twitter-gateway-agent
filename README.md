# Building a Twitter Gateway Agent

This guide will help you create an automated agent that monitors Twitter/X for relevant content based on your professional interests and notifies you only when something important is detected. This approach allows you to harness X's value for work while minimizing its drain on your time and energy.

## Overview

Based on Brian's suggestion, we'll build a system that:

1. Creates a Twitter list of accounts relevant to your work
2. Regularly fetches tweets from this list using the socialdata.tools API
3. Filters tweets based on engagement metrics to reduce noise
4. Uses an LLM to assess each tweet's relevance to your work
5. Generates a digest of the most relevant tweets and sends it to your preferred notification channel

## Step 1: Set Up Your Twitter List

1. **Create a List on Twitter:**

   - Log in to your Twitter/X account
   - Go to Lists → Create List
   - Name it something relevant to your work focus
   - Add accounts that frequently share content valuable to your work
   - Make note of the List ID (visible in the URL when viewing the list)

2. **Get Your List ID:**
   - When viewing your list on Twitter, the URL will look like: `https://twitter.com/i/lists/1234567890123456789`
   - The number at the end is your List ID

## Step 2: Get API Access

1. **Sign up for socialdata.tools:**
   - Go to [socialdata.tools](https://socialdata.tools) and create an account
   - Purchase API credits (they offer various plans)
   - Get your API key from your account dashboard

## Step 3: Build the Tweet Fetching Script

Create a Python script to fetch tweets from your list:

```python
import requests
import os
from datetime import datetime, timedelta
import json

# Configuration
API_KEY = "your_socialdata_tools_api_key"
LIST_ID = "your_twitter_list_id"
MIN_LIKES = 10  # Adjust based on your list's typical engagement
HEADERS = {
    "Authorization": f"Bearer {API_KEY}",
    "Accept": "application/json"
}

def fetch_list_tweets():
    """Fetch recent tweets from your Twitter list."""
    url = f"https://api.socialdata.tools/twitter/list/{LIST_ID}/tweets"

    response = requests.get(url, headers=HEADERS)

    if response.status_code == 200:
        data = response.json()
        return data["tweets"]
    else:
        print(f"Error fetching tweets: {response.status_code}")
        print(response.text)
        return []

def filter_tweets_by_engagement(tweets, min_likes=MIN_LIKES):
    """Filter tweets based on minimum engagement to reduce noise."""
    return [
        tweet for tweet in tweets
        if tweet["favorite_count"] >= min_likes
    ]

def save_tweets_for_analysis(tweets):
    """Save filtered tweets to a JSON file for LLM analysis."""
    with open("tweets_for_analysis.json", "w") as f:
        json.dump(tweets, f, indent=2)

    print(f"Saved {len(tweets)} tweets for analysis")

if __name__ == "__main__":
    # Fetch tweets from list
    all_tweets = fetch_list_tweets()
    print(f"Fetched {len(all_tweets)} tweets from list")

    # Filter by engagement
    filtered_tweets = filter_tweets_by_engagement(all_tweets)
    print(f"Filtered to {len(filtered_tweets)} tweets with {MIN_LIKES}+ likes")

    # Save for analysis
    save_tweets_for_analysis(filtered_tweets)
```

## Step 4: Build the LLM Relevance Assessment

Now create a script to analyze tweets with an LLM:

```python
import json
import os
from openai import OpenAI

# Configuration
OPENAI_API_KEY = "your_openai_api_key"
WORK_DESCRIPTION = """
Provide a detailed description of your work interests, projects, and topics
that would make a tweet relevant to you. The more specific, the better.
For example: "I'm working on AI agents for productivity, specifically
focusing on automating knowledge work tasks and reducing digital distraction."
"""

client = OpenAI(api_key=OPENAI_API_KEY)

def load_tweets():
    """Load tweets saved for analysis."""
    with open("tweets_for_analysis.json", "r") as f:
        return json.load(f)

def analyze_tweet_relevance(tweet):
    """Use LLM to analyze if a tweet is relevant to your work interests."""
    tweet_text = tweet["full_text"]
    username = tweet["user"]["screen_name"]
    engagement = f"Likes: {tweet['favorite_count']}, Retweets: {tweet['retweet_count']}"
    tweet_url = f"https://twitter.com/{username}/status/{tweet['id_str']}"

    prompt = f"""
    Analyze if the following tweet is highly relevant to my work:

    TWEET: "{tweet_text}"
    By: @{username}
    Engagement: {engagement}

    MY WORK INTERESTS:
    {WORK_DESCRIPTION}

    Rate this tweet's relevance to my work on a scale of 1-10, where:
    1-3: Not relevant
    4-6: Somewhat relevant
    7-10: Highly relevant

    First provide the numerical score, then a brief explanation.
    """

    response = client.chat.completions.create(
        model="gpt-4o",  # Or another suitable model
        messages=[
            {"role": "system", "content": "You evaluate content relevance for busy professionals."},
            {"role": "user", "content": prompt}
        ],
        temperature=0.2
    )

    analysis = response.choices[0].message.content

    # Extract score from response (assuming format like "Score: 8/10")
    try:
        score_line = analysis.split('\n')[0]
        score = int(score_line.strip().split(':')[1].strip().split('/')[0])
    except:
        # Fallback if parsing fails
        score = 0

    return {
        "tweet": tweet,
        "tweet_url": tweet_url,
        "score": score,
        "analysis": analysis
    }

def generate_digest(analyses):
    """Generate a digest of the most relevant tweets."""
    # Sort by relevance score (highest first)
    relevant_tweets = sorted(
        [a for a in analyses if a["score"] >= 7],  # Only include highly relevant tweets
        key=lambda x: x["score"],
        reverse=True
    )

    if not relevant_tweets:
        return "No highly relevant tweets found in the latest batch."

    digest = "# Relevant Twitter Content Digest\n\n"

    for item in relevant_tweets:
        tweet = item["tweet"]
        username = tweet["user"]["screen_name"]
        digest += f"## @{username}: {item['score']}/10 Relevance\n\n"
        digest += f"{tweet['full_text']}\n\n"
        digest += f"[View Tweet]({item['tweet_url']})\n\n"
        digest += "---\n\n"

    return digest

if __name__ == "__main__":
    # Load tweets to analyze
    tweets = load_tweets()
    print(f"Analyzing {len(tweets)} tweets for relevance...")

    # Analyze each tweet
    analyses = [analyze_tweet_relevance(tweet) for tweet in tweets]

    # Generate digest of relevant tweets
    digest = generate_digest(analyses)

    # Save digest to file
    with open("twitter_digest.md", "w") as f:
        f.write(digest)

    print("Digest generated and saved to twitter_digest.md")
```

## Step 5: Set Up Notifications

Now let's create a script to send the digest to your preferred channel:

```python
import requests
import os

# Configuration - choose one method
DISCORD_WEBHOOK_URL = "your_discord_webhook_url"  # For Discord notifications
EMAIL_API_KEY = "your_email_service_api_key"      # For email notifications

def send_to_discord(digest):
    """Send the digest to a Discord channel via webhook."""
    payload = {
        "content": "New Twitter Digest Available!",
        "embeds": [
            {
                "title": "Twitter Content Digest",
                "description": digest[:2000] + "..." if len(digest) > 2000 else digest,
                "color": 3447003  # Blue color
            }
        ]
    }

    response = requests.post(DISCORD_WEBHOOK_URL, json=payload)

    if response.status_code == 204:
        print("Digest sent to Discord successfully!")
    else:
        print(f"Error sending to Discord: {response.status_code}")
        print(response.text)

def read_digest():
    """Read the generated digest file."""
    with open("twitter_digest.md", "r") as f:
        return f.read()

if __name__ == "__main__":
    # Read the digest
    digest = read_digest()

    # Send to notification channel (Discord in this example)
    send_to_discord(digest)
```

## Step 6: Automate the Workflow

Create a main script to run the entire process:

```python
import os
import subprocess
import time

def run_full_workflow():
    """Run the full Twitter monitoring workflow."""
    print("Starting Twitter Gateway workflow...")

    # Step 1: Fetch and filter tweets
    print("\n--- Fetching Tweets ---")
    subprocess.run(["python", "fetch_tweets.py"])

    # Step 2: Analyze tweets with LLM
    print("\n--- Analyzing Tweets ---")
    subprocess.run(["python", "analyze_tweets.py"])

    # Step 3: Send notifications if digest was generated
    if os.path.exists("twitter_digest.md"):
        print("\n--- Sending Notification ---")
        subprocess.run(["python", "send_notification.py"])
    else:
        print("\nNo digest was generated - no relevant tweets found.")

    print("\nWorkflow complete!")

if __name__ == "__main__":
    run_full_workflow()
```

## Step 7: Schedule Regular Runs

Set up your script to run on a schedule:

### On Linux/Mac (using cron):

1. Open your crontab:

   ```
   crontab -e
   ```

2. Add a line to run it every 3 hours (adjust as needed):
   ```
   0 */3 * * * cd /path/to/your/project && python run_workflow.py >> twitter_gateway.log 2>&1
   ```

### On Windows (using Task Scheduler):

1. Open Task Scheduler
2. Create a Basic Task
3. Set the trigger to run daily, with a recurrence of every 3 hours
4. Set the action to start a program: `python.exe` with argument `run_workflow.py`
5. Set the start in location to your project directory

## Customization Options

1. **Filtering Options**: Adjust the `MIN_LIKES` threshold based on the typical engagement in your Twitter list.

2. **Relevance Criteria**: Refine the `WORK_DESCRIPTION` to better capture what makes content relevant to you.

3. **Schedule Frequency**: Change the cron/task schedule to check more or less frequently.

4. **Multiple Lists**: Run the script for multiple different Twitter lists focused on different aspects of your work.

5. **Notification Channels**: Modify the notification script to send to other platforms like Slack, Telegram, or email.

## Final Tips

1. **Start with a small list** of very relevant accounts and expand gradually.

2. **Adjust thresholds over time** to find the right balance between missing important content and receiving too many notifications.

3. **Regularly update your work interests** in the script as your professional focus evolves.

4. **Consider adding a "manual override"** feature that lets you receive notifications about tweets from specific high-priority accounts regardless of their engagement metrics.

With this system in place, you can stay informed about highly relevant Twitter content without the constant distraction of the platform itself!

"""
