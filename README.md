# Data_Analyst

* PY Insights (Power of You) Data Analyst pre-task.

* Tools i have use this task 
Python
pandas
numpy
matplotlib
seaborn
urllib.parse (urlparse)
Jupyter Notebook (inside VS Code)

* To read and understand the data i have used pandas

Columns were named Summary, Unnamed: 1, Unnamed: 9 and more  not meaningful. The file actually contained multiple sections. Summary rows at the top (OrgId, ParticipantId, etc.). Blank rows. A row with "Browsing" label. Then the real header row with columns like. OrgId, ParticipantId, DeviceId, url, eventtimeutc, transition, title, visitId, referringVisitId, eventtime. Then the browsing logs below it. Conclusion: we needed to extract the real header and rebuild the dataframe.

* Clean & Reshape the Data (Extract true header)
We identified the row that contains the true header (where the last column is eventtime), extracted it as column names, and created a clean dataframe from the rows below

header_index = df[df['Unnamed: 9'] == 'eventtime'].index[0]

true_header = df.iloc[header_index].values

df_clean = df.iloc[header_index + 1:].copy()
df_clean.columns = true_header
df_clean = df_clean.reset_index(drop=True)

df_clean.head()
df_clean.info()

 Why we did it Real-world CSVs are often messy They may contain metadata, summaries, and multiple header rows.If we don’t fix the header, all later analysis (groupby, filtering) becomes hard or wrong. So this step Removes non-data rows (summary/empty rows). Gives us correct column names Leaves us with only actual browsing events.

* Parse Timestamps & Create Time Features
df_clean['eventtime'] = pd.to_datetime(df_clean['eventtime'], errors='coerce')
df_clean['eventtimeutc'] = pd.to_datetime(df_clean['eventtimeutc'], errors='coerce')

df_clean['date'] = df_clean['eventtime'].dt.date
df_clean['hour'] = df_clean['eventtime'].dt.hour
df_clean['dayofweek'] = df_clean['eventtime'].dt.day_name()

Why we did it
To analyze behavior over time, we need time columns in proper datetime format, not just strings.

We created:
date for daily activity trends (number of visits per day)
hour for time-of-day patterns (which hours are most active)
dayofweek for weekday patterns (which days are busiest)
These features power multiple visualizations later (line chart, heatmap, and many more).

What we understood
Now we can answer questions like: On which days was the user most active? Are they more active on weekdays or weekends? Do they browse mostly in the morning, afternoon, or night?

* Extract Domain from URL
from urllib.parse import urlparse

def extract_domain(url):
    try:
        domain = urlparse(url).netloc
        return domain.replace("www.", "")
    except:
        return None

df_clean['domain'] = df_clean['url'].apply(extract_domain)

Why we did it.

Raw URLs are too detailed and messy.
For behavior analysis, it is more useful to know which website the user visited:https://www.google.com/search? google.com https://upwork.com/jobs/... → upwork.com

This lets us:Count visits per domain.Identify top sites (google, upwork, etc.).Map domains to behavioral categories (e.g., jobs, travel, work tools).

df_clean[['url','domain']].head(10)
df_clean['domain'].value_counts().head(15)

What we saw
Using value_counts() on domain, we saw google.com – 1328 visits, upwork.com – 473, mail.google.com, sharepoint, loopnet, aws console, amazon.in, zipair.net, etc.This already hints at job search, work tools, travel, and shopping behavior.

* Map Domains to Behavioral Categories
def categorize_domain(domain):
    if domain is None:
        return "Other"
    d = domain.lower()
    
    if "upwork" in d or "wellfound" in d or "taskrabbit" in d or "linkedin" in d:
        return "Jobs & Freelancing"
    if "amazon." in d or "flipkart" in d or "instacart" in d:
        return "E-commerce"
    if "zipair" in d or "airasia" in d or "britishairways" in d:
        return "Travel"
    if ("sharepoint" in d or "gitlab" in d or "xero" in d 
        or "aws.amazon" in d or "console.aws.amazon" in d
        or "py-insights.com" in d or "pyinsights" in d
        or "teams.microsoft" in d or "onedrive.live.com" in d
        or "login.microsoftonline" in d or "app.pinecone.io" in d):
        return "Work & Tools"
    if ("google.com" in d or "chromewebstore.google.com" in d 
        or "accounts.google.com" in d or "mail.google.com" in d):
        return "Search & Research"
    if "facebook.com" in d or "instagram.com" in d or "x.com" in d or "twitter.com" in d:
        return "Social"
    return "Other"

df_clean["category"] = df_clean["domain"].apply(categorize_domain)

Why we did it 

we grouped domains so we can talk in human terms, e.g.: Jobs & Freelancing, Work & Tools, Search & Research, E-commerce, Travel, Social, Other

What we learned category.value_counts() gave us: Search & Research – 1704 Other – 1566 Jobs & Freelancing – 667 Work & Tools – 492 E-commerce – 365 Social – 163 Travel – 147

This tells us where their time goes at a high level.

* Visits by Behavioral Category
import matplotlib.pyplot as plt

category_counts = df_clean["category"].value_counts()

plt.figure(figsize=(10,4))
category_counts.plot(kind="bar", color="skyblue")
plt.title("Visits by Behavioral Category")
plt.xlabel("Category")
plt.ylabel("Number of Visits")
plt.xticks(rotation=45, ha="right")
plt.tight_layout()
plt.show()

Why we used this code/tool A bar chart is ideal for comparing totals across categories. This gives a clear visual of which behavior types dominate. matplotlib is simple, widely used, and integrates well with pandas.

What we understood
Search & Research dominates (most visits).Strong activity in Jobs & Freelancing and Work & Tools. Moderate E-commerce & Travel. Limited Social media.
This supports the story of a work- and job-search-focused user.

* Daily Browsing Activity Over Time
daily_counts = df_clean.groupby("date").size()

plt.figure(figsize=(12,4))
daily_counts.plot(kind="line", marker="o")
plt.title("Daily Browsing Activity Over Time")
plt.xlabel("Date")
plt.ylabel("Number of Visits")
plt.xticks(rotation=45)
plt.tight_layout()
plt.show()

Why we used this code/tool
groupby("date").size() counts visits per day. A line plot is ideal for showing change across time. This reveals busy vs quiet days, trends, and spikes.

What we understood
Some days (like ~Jan 28 and ~Feb 19) are very intense (500+ visits). Weekends generally show lower activity. Suggests bursts of focused work/research on specific days.

This helps build a narrative around productivity cycles.

* Heatmap of Hour vs Day-of-Week
import seaborn as sns

pivot = df_clean.pivot_table(
    index="dayofweek",
    columns="hour",
    values="url",
    aggfunc="count"
).fillna(0)

day_order = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday"]
pivot = pivot.reindex(day_order)

plt.figure(figsize=(14,6))
sns.heatmap(pivot, cmap="Blues", linewidths=0.5)
plt.title("Heatmap of Browsing Activity by Hour & Day of Week")
plt.xlabel("Hour of Day")
plt.ylabel("Day of Week")
plt.tight_layout()
plt.show()

Why we used this code/tool A heatmap is perfect to visualize intensity across two dimensions Rows → Day of week Columns → Hour of day Color → Number of visits pivot_table prepares the 2D matrix. seaborn.heatmap visualizes it nicely.

What we understood
User is most active in evenings (20:00–23:00). Peak days: Tuesday, Wednesday, Thursday. Saturday is relatively quiet.

This shows a pattern of weekday evening-heavy browsing, likely after typical work hours

* Top 15 Most Visited Domains
top_domains = df_clean['domain'].value_counts().head(15)

plt.figure(figsize=(12,5))
top_domains.plot(kind='bar', color="lightgreen")
plt.title("Top 15 Most Visited Domains")
plt.xlabel("Domain")
plt.ylabel("Number of Visits")
plt.xticks(rotation=45, ha="right")
plt.tight_layout()
plt.show()

Why we used this code/tool
We wanted to see specific sites, not just categories A bar chart of top 15 domains gives a sharp view of priorities. This complements the category-level view.

What we understood
google.com is the top site (search/research). upwork.com, wellfound.com, taskrabbit.com → strong job/freelance focus. pyinsightscom.sharepoint.com, aws console, gitlab.com, xero.com → work and technical tools.amazon.in, zipair.net → personal shopping and travel.

This confirms a mix of professional, job-search, and personal activity.


* Distribution of Transition Types
transition_counts = df_clean['transition'].value_counts()

plt.figure(figsize=(10,4))
transition_counts.plot(kind='bar', color="coral")
plt.title("Distribution of Transition Types")
plt.xlabel("Transition Type")
plt.ylabel("Number of Visits")
plt.xticks(rotation=45)
plt.tight_layout()
plt.show()

transition_counts

Why we used this code/tool
The transition column tells us how the user reached each page:link – clicking a hyperlink reload – refreshing the page generated – browser-generated pages form_submit – form submissions (search, login) typed – manually typed URL, A bar chart shows which navigation types are most common → reveals browsing style.

What we understood
link dominates (~4098 events) → mostly exploratory click-based browsing. Non-trivial reload and generated → repeated checking of content (dashboards, feeds, results). form_submit and typed show intentional actions (searching, going to specific sites).

This shows a mix of exploration and task-driven navigation.


* Summary for README
Putting all steps together, we understand that: The user’s browsing is heavily focused on Search & Research (Google and related services).

They show clear job & freelancing intent, with frequent visits to Upwork, Wellfound, TaskRabbit.

They regularly use work tools & technical platforms like SharePoint, GitLab, AWS Console, Xero.

Browsing peaks in the evenings, mainly on weekdays, especially mid-week.

There is some E-commerce and Travel activity, and low but present social media use.

Navigation is mostly via link clicks, with some deliberate searches (form_submit) and typed URLs.

This end-to-end flow from raw messy CSV to clean insights demonstrates:

Data cleaning with pandas

Feature engineering

Behavioral categorization

Visualization skills

Clear, business-oriented interpretation

 