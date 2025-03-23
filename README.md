## Social Media Forensic Tool For Cybercrime Investigation And Analysis Using AI
With the rise of cybercrimes such as identity theft, misinformation, cyberbullying, and online fraud, law enforcement agencies require advanced tools to analyze and investigate social media activities. The Social Media Forensic Tool for Cybercrime Investigation and Analysis Using AI is designed to extract, analyze, and visualize social media data to assist forensic experts in cyber investigations. By leveraging AI-driven techniques, this tool can detect suspicious activities, analyze patterns, and generate detailed reports to aid in cybercrime prevention and resolution.
## About
This project is a Streamlit-based AI-powered web application that retrieves and processes public social media data from platforms like Instagram and YouTube. It integrates web scraping, machine learning, and API-based data extraction to provide investigators with detailed insights into user activities, media interactions, and engagement metrics. The system employs natural language processing (NLP), anomaly detection, and deep learning algorithms to identify potential cyber threats.
## Key Features
1.Automated Social Media Data Extraction: Retrieves user profiles, posts, comments, and engagement metrics using APIs and web scraping.

2.AI-Powered Cybercrime Detection: Uses machine learning models to identify suspicious patterns, fake profiles, and hate speech.

3.Sentiment and Behavioral Analysis: Employs NLP to analyze user intent, sentiment, and interactions.

4.Visualization and Reporting: Generates interactive graphs, trend analysis reports, and dashboards for forensic examination.

5.Real-Time Monitoring: Provides live tracking of social media trends and flagged activities.

6.User-Friendly Interface: Developed using Streamlit, making it accessible for forensic analysts and law enforcement.
## System Requirements
### Hardware Requirements
Minimum 8GB RAM (Recommended: 16GB+)

Intel i5/i7 or AMD Ryzen 5/7 processor

SSD (Minimum 256GB for faster processing)

Stable internet connection for API requests

### Software Requirements
Operating System: Windows 10/11, macOS, or Linux

Programming Language: Python 3.8+

Libraries & Frameworks:

Streamlit (for UI development)

Scrapy / BeautifulSoup (for web scraping)

TensorFlow / PyTorch (for AI analysis)

Scikit-learn (for machine learning)

Matplotlib & Seaborn (for data visualization)
## Program
```
import streamlit as st
import json
import httpx
from PIL import Image
from io import BytesIO
import time
import datetime
import random
from typing import Dict, List, Tuple, Union, Any
import googleapiclient.discovery
import googleapiclient.errors

# Set page configuration
st.set_page_config(
    page_title="Social Media Scraper",
    page_icon="🔍",
    layout="wide"
)

# Session state initialization
if 'last_request_time' not in st.session_state:
    st.session_state.last_request_time = 0
if 'cached_images' not in st.session_state:
    st.session_state.cached_images = {}

# Constants
YOUTUBE_API_KEY = ""  # Replace with your actual API key
INSTAGRAM_COOKIE = ""  # Replace with your valid sessionid cookie

def create_http_client():
    """Create an HTTP client with random user agent and session cookie"""
    user_agents = [
        "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
        "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Safari/605.1.15",
        "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
    ]
    headers = {
        "User-Agent": random.choice(user_agents),
        "Accept-Language": "en-US,en;q=0.9",
        "Accept-Encoding": "gzip, deflate, br",
        "Accept": "*/*",
        "X-IG-App-ID": "936619743392459",
    }
    if INSTAGRAM_COOKIE:
        headers["Cookie"] = f"sessionid={INSTAGRAM_COOKIE}"
    return httpx.Client(headers=headers, timeout=30.0)

client = create_http_client()

def rate_limit_request() -> None:
    """Implement rate limiting"""
    current_time = time.time()
    elapsed = current_time - st.session_state.last_request_time
    if elapsed < 2:
        time.sleep(2 - elapsed)
    time.sleep(random.uniform(0.5, 1.5))
    st.session_state.last_request_time = time.time()

def load_image_from_url(url: str, size: str = "Medium") -> Image.Image:
    """Load an image from a URL with caching"""
    width_map = {"Small": 150, "Medium": 250, "Large": 350}
    cache_key = f"{url}_{width_map[size]}"
    if cache_key in st.session_state.cached_images:
        return st.session_state.cached_images[cache_key]
    try:
        rate_limit_request()
        response = client.get(url)
        if response.status_code == 200:
            img = Image.open(BytesIO(response.content))
            st.session_state.cached_images[cache_key] = img
            return img
        else:
            st.warning(f"Failed to load image from {url}: Status code {response.status_code}")
            return None
    except Exception as e:
        st.warning(f"Error loading image from {url}: {str(e)}")
        return None

###################
### INSTAGRAM ####
###################

@st.cache_data(ttl=3600)
def scrape_instagram_user(username: str) -> Tuple[Union[Dict, str], List[Dict], List[Dict]]:
    """Scrape Instagram user's data and media with session authentication"""
    try:
        # Step 1: Get user ID and basic info
        rate_limit_request()
        profile_url = f"https://i.instagram.com/api/v1/users/web_profile_info/?username={username}"
        profile_result = client.get(profile_url)
        
        if profile_result.status_code == 401:
            return "Authentication failed (401). Check your session cookie.", [], []
        elif profile_result.status_code != 200:
            return f"Failed to retrieve profile data: {profile_result.status_code}", [], []
        
        profile_data = profile_result.json()
        user_info = profile_data.get("data", {}).get("user", {})
        if not user_info:
            return "User not found or private", [], []

        user_id = user_info.get("id", "N/A")
        if user_id == "N/A":
            return "Could not retrieve user ID", [], []

        user = {
            "Username": user_info.get("username", "N/A"),
            "Full Name": user_info.get("full_name", "N/A"),
            "ID": user_id,
            "Category": user_info.get("category_name", "N/A"),
            "Business Category": user_info.get("business_category_name", "N/A"),
            "Phone": user_info.get("business_phone_number", "N/A"),
            "Email": user_info.get("business_email", "N/A"),
            "Biography": user_info.get("biography", "N/A"),
            "Bio Links": [link.get("url") for link in user_info.get("bio_links", []) if link.get("url")],
            "Homepage": user_info.get("external_url", "N/A"),
            "Followers": f"{user_info.get('edge_followed_by', {}).get('count', 0):,}",
            "Following": f"{user_info.get('edge_follow', {}).get('count', 0):,}",
            "Facebook ID": user_info.get("fbid", "N/A"),
            "Is Private": user_info.get("is_private", False),
            "Is Verified": user_info.get("is_verified", False),
            "Profile Image": user_info.get("profile_pic_url_hd", user_info.get("profile_pic_url", "N/A")),
            "Video Count": user_info.get("edge_felix_video_timeline", {}).get("count", 0),
            "Image Count": user_info.get("edge_owner_to_timeline_media", {}).get("count", 0),
            "Saved Count": user_info.get("edge_saved_media", {}).get("count", 0),
            "Collections Count": user_info.get("edge_saved_collections", {}).get("count", 0),
            "Related Profiles": [profile.get("node", {}).get("username", "N/A") for profile in user_info.get("edge_related_profiles", {}).get("edges", [])],
            "Is Business": user_info.get("is_business_account", False),
            "Joined Recently": user_info.get("is_joined_recently", False),
            "Highlight Reel Count": user_info.get("highlight_reel_count", 0),
        }

        # Step 2: Get media data (default to 12 items)
        rate_limit_request()
        media_url = f"https://i.instagram.com/api/v1/feed/user/{user_id}/?count=12"
        media_result = client.get(media_url)
        
        if media_result.status_code != 200:
            st.warning(f"Failed to retrieve media data: {media_result.status_code}. Media may not be visible.")
            return user, [], []

        media_data = media_result.json()
        items = media_data.get("items", [])
        
        if not items:
            st.info("No media items found. Account may be private or empty.")
            return user, [], []

        videos = []
        images = []
        for item in items:
            is_video = item.get("media_type") == 2 or item.get("is_video", False)
            timestamp = item.get("taken_at", 0) or item.get("device_timestamp", 0)
            caption = item.get("caption", {}).get("text", "N/A") if item.get("caption") else "N/A"
            media_info = {
                "ID": item.get("id", "N/A"),
                "Title": caption[:50] if caption != "N/A" else "N/A",
                "Shortcode": item.get("code", "N/A"),
                "Thumbnail": item.get("image_versions2", {}).get("candidates", [{}])[0].get("url", "N/A"),
                "URL": item.get("video_versions", [{}])[0].get("url", "N/A") if is_video else item.get("image_versions2", {}).get("candidates", [{}])[0].get("url", "N/A"),
                "Tagged": [user.get("username", "N/A") for user in item.get("usertags", {}).get("in", [])],
                "Captions": [caption] if caption != "N/A" else [],
                "Comments Count": item.get("comment_count", 0),
                "Comments Disabled": item.get("comments_disabled", False),
                "Taken At": timestamp,
                "Taken At Formatted": datetime.datetime.fromtimestamp(timestamp).strftime('%Y-%m-%d %H:%M:%S') if timestamp else "N/A",
                "Likes": item.get("like_count", 0),
                "Location": item.get("location", {}).get("name", "N/A") if item.get("location") else "N/A",
                "Duration": item.get("video_duration", "N/A") if is_video else "N/A",
                "Is Sponsored": item.get("is_ad", False),
                "Post URL": f"https://www.instagram.com/p/{item.get('code', 'N/A')}/",
                "Is Video": is_video,
                "Accessibility Caption": item.get("accessibility_caption", "N/A") if not is_video else "N/A",
            }
            if is_video:
                videos.append(media_info)
            else:
                images.append(media_info)

        return user, videos, images

    except Exception as e:
        return f"Error: {str(e)}", [], []

def display_instagram_user_info(user_info: Union[Dict, str]) -> None:
    """Display Instagram user info"""
    st.subheader("Instagram Profile")
    if isinstance(user_info, str):
        st.error(user_info)
    else:
        with st.container():
            col1, col2 = st.columns([3, 2])
            with col1:
                with st.container():
                    st.markdown(f"### {user_info.get('Full Name')} (@{user_info.get('Username')})")
                    if user_info.get('Is Verified'):
                        st.markdown("✅ Verified Account")
                    if user_info.get('Is Private'):
                        st.markdown("🔒 Private Account")
                    if user_info.get('Is Business'):
                        st.markdown("💼 Business Account")
                    st.markdown("#### Stats")
                    stat_cols = st.columns(3)
                    with stat_cols[0]:
                        st.metric("Followers", user_info.get('Followers'))
                    with stat_cols[1]:
                        st.metric("Following", user_info.get('Following'))
                    with stat_cols[2]:
                        st.metric("Posts", f"{int(user_info.get('Video Count', 0)) + int(user_info.get('Image Count', 0)):,}")
                    st.markdown("#### Bio")
                    st.write(user_info.get('Biography'))
                    if user_info.get('Bio Links') and user_info.get('Bio Links')[0] != "N/A":
                        st.markdown("#### Links")
                        for url in user_info.get('Bio Links', []):
                            st.markdown(f"[🔗 {url}]({url})")
                    if user_info.get('Homepage') and user_info.get('Homepage') != "N/A":
                        st.markdown(f"[🌐 {user_info.get('Homepage')}]({user_info.get('Homepage')})")
                    if user_info.get('Category') != "N/A" or user_info.get('Business Category') != "N/A":
                        st.markdown("#### Business Information")
                        if user_info.get('Category') != "N/A":
                            st.write(f"Category: {user_info.get('Category')}")
                        if user_info.get('Business Category') != "N/A":
                            st.write(f"Business Category: {user_info.get('Business Category')}")
                        if user_info.get('Phone') != "N/A":
                            st.write(f"Phone: {user_info.get('Phone')}")
                        if user_info.get('Email') != "N/A":
                            st.write(f"Email: {user_info.get('Email')}")
                    with st.expander("Additional Info"):
                        st.write(f"ID: {user_info.get('ID')}")
                        st.write(f"Video Count: {user_info.get('Video Count')}")
                        st.write(f"Image Count: {user_info.get('Image Count')}")
                        st.write(f"Saved Count: {user_info.get('Saved Count')}")
                        st.write(f"Collections Count: {user_info.get('Collections Count')}")
                        st.write(f"Highlight Reel Count: {user_info.get('Highlight Reel Count')}")
                        st.write(f"Facebook ID: {user_info.get('Facebook ID')}")
                        st.write(f"Joined Recently: {'Yes' if user_info.get('Joined Recently') else 'No'}")
                    if user_info.get('Related Profiles') and user_info.get('Related Profiles')[0] != "N/A":
                        with st.expander("Related Profiles"):
                            for profile in user_info.get('Related Profiles', []):
                                st.write(f"@{profile}")
                st.download_button(
                    "Download User Data",
                    json.dumps(user_info, indent=4),
                    f"{user_info.get('Username')}_profile.json",
                    key=f"insta_profile_{user_info.get('Username')}"
                )
            with col2:
                if user_info.get('Profile Image') != "N/A":
                    img = load_image_from_url(user_info.get('Profile Image'))
                    if img:
                        st.image(img, caption="Profile Picture", use_column_width=True)

def display_instagram_media(media_list: List[Dict], media_type: str, username: str) -> None:
    """Display Instagram media"""
    if not media_list:
        st.info(f"No {media_type}s found or account may be private.")
        return

    state_key_prefix = f"insta_{username}_{media_type}"
    if f"{state_key_prefix}_sort" not in st.session_state:
        st.session_state[f"{state_key_prefix}_sort"] = "Likes"
    if f"{state_key_prefix}_thumb" not in st.session_state:
        st.session_state[f"{state_key_prefix}_thumb"] = "Medium"
    if f"{state_key_prefix}_view" not in st.session_state:
        st.session_state[f"{state_key_prefix}_view"] = "Grid"

    with st.container():
        st.subheader(f"{media_type.capitalize()}s")
        sort_options = {
            "Likes": lambda x: x.get('Likes', 0),
            "Comments": lambda x: x.get('Comments Count', 0),
            "Date (newest first)": lambda x: x.get('Taken At', 0),
            "Date (oldest first)": lambda x: -x.get('Taken At', 0) if x.get('Taken At') not in (0, "N/A") else 0
        }

        def update_sort():
            st.session_state[f"{state_key_prefix}_sort"] = st.session_state[f"{state_key_prefix}_sort_select"]
        def update_thumb():
            st.session_state[f"{state_key_prefix}_thumb"] = st.session_state[f"{state_key_prefix}_thumb_select"]
        def update_view():
            st.session_state[f"{state_key_prefix}_view"] = st.session_state[f"{state_key_prefix}_view_select"]

        sort_by = st.selectbox(
            f"Sort {media_type}s by:",
            list(sort_options.keys()),
            key=f"{state_key_prefix}_sort_select",
            index=list(sort_options.keys()).index(st.session_state[f"{state_key_prefix}_sort"]),
            on_change=update_sort
        )
        display_cols = st.columns([1, 1])
        with display_cols[0]:
            thumb_options = ["Small", "Medium", "Large"]
            thumbnail_size = st.radio(
                "Thumbnail Size:",
                thumb_options,
                key=f"{state_key_prefix}_thumb_select",
                index=thumb_options.index(st.session_state[f"{state_key_prefix}_thumb"]),
                on_change=update_thumb
            )
        with display_cols[1]:
            view_options = ["Grid", "List"]
            view_mode = st.radio(
                "View Mode:",
                view_options,
                key=f"{state_key_prefix}_view_select",
                index=view_options.index(st.session_state[f"{state_key_prefix}_view"]),
                on_change=update_view
            )

        width = {"Small": 150, "Medium": 250, "Large": 350}[thumbnail_size]
        sorted_media = sorted(media_list, key=sort_options[sort_by], reverse=sort_by != "Date (oldest first)")

        if view_mode == "Grid":
            col_count = 3 if thumbnail_size == "Small" else (2 if thumbnail_size == "Medium" else 1)
            grid_cols = st.columns(col_count)
            for i, media in enumerate(sorted_media):
                col_idx = i % col_count
                with grid_cols[col_idx]:
                    with st.container():
                        if media.get("Is Video") and media.get("URL") != "N/A":
                            st.video(media.get("URL"))
                        elif media.get("Thumbnail") != "N/A":
                            img = load_image_from_url(media.get("Thumbnail"), thumbnail_size)
                            if img:
                                st.image(img, width=width)
                            else:
                                st.warning("Image unavailable")
                        st.markdown(f"Likes: {media.get('Likes', 0):,} | Comments: {media.get('Comments Count', 0):,}")
                        st.markdown(f"Posted: {media.get('Taken At Formatted', 'N/A')}")
                        captions = media.get('Captions', [])
                        if captions and captions[0] != "N/A":
                            caption = captions[0]
                            st.markdown(f"{caption[:100]}..." if len(caption) > 100 else caption)
                        st.markdown(f"[View on Instagram]({media.get('Post URL')})")
        else:
            for media in sorted_media:
                with st.expander(f"{media.get('Taken At Formatted', 'Post')}"):
                    cols = st.columns([1, 3])
                    with cols[0]:
                        if media.get("Is Video") and media.get("URL") != "N/A":
                            st.video(media.get("URL"))
                        elif media.get("Thumbnail") != "N/A":
                            img = load_image_from_url(media.get("Thumbnail"), thumbnail_size)
                            if img:
                                st.image(img, width=width)
                            else:
                                st.warning("Image unavailable")
                    with cols[1]:
                        st.write(f"ID: {media.get('ID')}")
                        st.write(f"Shortcode: {media.get('Shortcode')}")
                        st.write(f"Likes: {media.get('Likes', 0):,}")
                        st.write(f"Comments: {media.get('Comments Count', 0):,}")
                        if media.get('Location') != "N/A":
                            st.write(f"Location: {media.get('Location')}")
                        tagged_users = media.get('Tagged', [])
                        if tagged_users and tagged_users[0] != "N/A":
                            st.write(f"Tagged Users: {', '.join('@' + tag for tag in tagged_users)}")
                        st.write(f"Posted: {media.get('Taken At Formatted', 'N/A')}")
                        if media.get('Duration') not in (0, "N/A"):
                            st.write(f"Duration: {media.get('Duration')} seconds")
                        if media.get('Is Sponsored'):
                            st.write("Sponsored: Yes")
                        captions = media.get('Captions', [])
                        if captions and captions[0] != "N/A":
                            st.write("Caption:")
                            st.write(captions[0])
                        st.markdown(f"[View on Instagram]({media.get('Post URL')})")

        st.download_button(
            f"Download All {media_type.capitalize()} Data",
            json.dumps(media_list, indent=4),
            f"{username}_{media_type}_data.json",
            key=f"insta_{media_type}_{username}"
        )

###################
### YOUTUBE ####
###################

def initialize_youtube_service():
    """Initialize YouTube API"""
    try:
        return googleapiclient.discovery.build("youtube", "v3", developerKey=YOUTUBE_API_KEY)
    except Exception as e:
        st.error(f"Failed to initialize YouTube API: {str(e)}")
        return None

@st.cache_data(ttl=3600)
def scrape_youtube_channel(channel_name: str, num_items: int) -> Tuple[Union[Dict, str], List[Dict]]:
    """Scrape YouTube channel data"""
    youtube = initialize_youtube_service()
    if not youtube:
        return "API initialization failed", []

    try:
        rate_limit_request()
        search_response = youtube.search().list(q=channel_name, part="snippet", type="channel", maxResults=1).execute()
        if not search_response.get("items"):
            return "No channel found", []

        channel_id = search_response["items"][0]["id"]["channelId"]
        channel_response = youtube.channels().list(part="snippet,statistics,contentDetails", id=channel_id).execute()
        if not channel_response.get("items"):
            return "Channel details unavailable", []

        channel = channel_response["items"][0]
        channel_info = {
            "Channel ID": channel_id,
            "Title": channel["snippet"].get("title", "N/A"),
            "Description": channel["snippet"].get("description", "N/A"),
            "Custom URL": channel["snippet"].get("customUrl", "N/A"),
            "Published At": channel["snippet"].get("publishedAt", "N/A"),
            "Thumbnail": channel["snippet"].get("thumbnails", {}).get("high", {}).get("url", "N/A"),
            "Country": channel["snippet"].get("country", "N/A"),
            "Subscribers": f"{int(channel['statistics'].get('subscriberCount', 0)):,}",
            "Videos": f"{int(channel['statistics'].get('videoCount', 0)):,}",
            "Views": f"{int(channel['statistics'].get('viewCount', 0)):,}",
            "Hidden Subscribers": channel["statistics"].get("hiddenSubscriberCount", False),
            "Uploads Playlist": channel["contentDetails"]["relatedPlaylists"].get("uploads", "N/A"),
        }

        if channel_info["Uploads Playlist"] == "N/A":
            return channel_info, []

        rate_limit_request()
        playlist_response = youtube.playlistItems().list(
            part="snippet,contentDetails",
            playlistId=channel_info["Uploads Playlist"],
            maxResults=num_items
        ).execute()

        videos = []
        for item in playlist_response.get("items", []):
            video_id = item["contentDetails"]["videoId"]
            video_snippet = item["snippet"]
            rate_limit_request()
            video_response = youtube.videos().list(part="statistics,snippet", id=video_id).execute()
            video_stats = video_response["items"][0]["statistics"] if video_response.get("items") else {}
            video_snippet_stats = video_response["items"][0]["snippet"] if video_response.get("items") else {}
            published_at = video_snippet.get("publishedAt")
            if published_at and published_at != "N/A":
                published_at_formatted = datetime.datetime.fromisoformat(published_at.replace("Z", "+00:00")).strftime('%Y-%m-%d %H:%M:%S')
            else:
                published_at_formatted = "N/A"
            videos.append({
                "ID": video_id,
                "Title": video_snippet.get("title", "N/A"),
                "Thumbnail": video_snippet.get("thumbnails", {}).get("high", {}).get("url", "N/A"),
                "Description": video_snippet.get("description", "N/A"),
                "Published At": published_at,
                "Published At Formatted": published_at_formatted,
                "Likes": int(video_stats.get("likeCount", 0)),
                "Views": int(video_stats.get("viewCount", 0)),
                "Comments": int(video_stats.get("commentCount", 0)),
                "URL": f"https://www.youtube.com/watch?v={video_id}",
                "Tags": video_snippet_stats.get("tags", []),
                "Category": video_snippet_stats.get("categoryId", "N/A"),
            })

        return channel_info, videos
    except Exception as e:
        return f"Error: {str(e)}", [], []

def display_youtube_channel_info(channel_info: Union[Dict, str]) -> None:
    """Display YouTube channel info"""
    st.subheader("YouTube Channel")
    if isinstance(channel_info, str):
        st.error(channel_info)
    else:
        with st.container():
            col1, col2 = st.columns([3, 2])
            with col1:
                with st.container():
                    st.markdown(f"### {channel_info.get('Title')}")
                    if channel_info.get('Custom URL') != "N/A":
                        st.markdown(f"**Custom URL:** @{channel_info.get('Custom URL')}")
                    st.markdown("#### Stats")
                    stat_cols = st.columns(3)
                    with stat_cols[0]:
                        subs = "Hidden" if channel_info.get('Hidden Subscribers') else channel_info.get('Subscribers')
                        st.metric("Subscribers", subs)
                    with stat_cols[1]:
                        st.metric("Videos", channel_info.get('Videos'))
                    with stat_cols[2]:
                        st.metric("Views", channel_info.get('Views'))
                    st.markdown("#### Description")
                    st.write(channel_info.get('Description', 'No description available'))
                    if channel_info.get('Published At') != "N/A":
                        st.write(f"**Joined:** {channel_info.get('Published At')[:10]}")
                    with st.expander("Additional Info"):
                        st.write(f"Channel ID: {channel_info.get('Channel ID')}")
                        st.write(f"Country: {channel_info.get('Country')}")
                        st.write(f"Uploads Playlist ID: {channel_info.get('Uploads Playlist')}")
                st.download_button(
                    "Download Channel Data",
                    json.dumps(channel_info, indent=4),
                    f"{channel_info.get('Title')}_profile.json",
                    key=f"yt_channel_{channel_info.get('Channel ID')}"
                )
            with col2:
                if channel_info.get('Thumbnail') != "N/A":
                    img = load_image_from_url(channel_info.get('Thumbnail'))
                    if img:
                        st.image(img, caption="Channel Thumbnail", use_column_width=True)

def display_youtube_videos(video_list: List[Dict], channel_name: str) -> None:
    """Display YouTube videos"""
    if not video_list:
        st.info("No videos found or uploads playlist unavailable.")
        return

    state_key_prefix = f"yt_{channel_name}_videos"
    if f"{state_key_prefix}_sort" not in st.session_state:
        st.session_state[f"{state_key_prefix}_sort"] = "Likes"
    if f"{state_key_prefix}_thumb" not in st.session_state:
        st.session_state[f"{state_key_prefix}_thumb"] = "Medium"
    if f"{state_key_prefix}_view" not in st.session_state:
        st.session_state[f"{state_key_prefix}_view"] = "Grid"

    with st.container():
        st.subheader("Videos")
        sort_options = {
            "Likes": lambda x: x.get('Likes', 0),
            "Views": lambda x: x.get('Views', 0),
            "Comments": lambda x: x.get('Comments', 0),
            "Date (newest first)": lambda x: x.get('Published At', "N/A"),
            "Date (oldest first)": lambda x: -time.mktime(time.strptime(x.get('Published At', "1970-01-01T00:00:00Z")[:19], '%Y-%m-%dT%H:%M:%S')) if x.get('Published At') != "N/A" else 0
        }

        def update_sort():
            st.session_state[f"{state_key_prefix}_sort"] = st.session_state[f"{state_key_prefix}_sort_select"]
        def update_thumb():
            st.session_state[f"{state_key_prefix}_thumb"] = st.session_state[f"{state_key_prefix}_thumb_select"]
        def update_view():
            st.session_state[f"{state_key_prefix}_view"] = st.session_state[f"{state_key_prefix}_view_select"]

        sort_by = st.selectbox(
            "Sort videos by:",
            list(sort_options.keys()),
            key=f"{state_key_prefix}_sort_select",
            index=list(sort_options.keys()).index(st.session_state[f"{state_key_prefix}_sort"]),
            on_change=update_sort
        )
        display_cols = st.columns([1, 1])
        with display_cols[0]:
            thumb_options = ["Small", "Medium", "Large"]
            thumbnail_size = st.radio(
                "Thumbnail Size:",
                thumb_options,
                key=f"{state_key_prefix}_thumb_select",
                index=thumb_options.index(st.session_state[f"{state_key_prefix}_thumb"]),
                on_change=update_thumb
            )
        with display_cols[1]:
            view_options = ["Grid", "List"]
            view_mode = st.radio(
                "View Mode:",
                view_options,
                key=f"{state_key_prefix}_view_select",
                index=view_options.index(st.session_state[f"{state_key_prefix}_view"]),
                on_change=update_view
            )

        width = {"Small": 150, "Medium": 250, "Large": 350}[thumbnail_size]
        sorted_videos = sorted(video_list, key=sort_options[sort_by], reverse=sort_by != "Date (oldest first)")

        if view_mode == "Grid":
            col_count = 3 if thumbnail_size == "Small" else (2 if thumbnail_size == "Medium" else 1)
            grid_cols = st.columns(col_count)
            for i, video in enumerate(sorted_videos):
                col_idx = i % col_count
                with grid_cols[col_idx]:
                    with st.container():
                        if video.get("URL") != "N/A":
                            st.video(video.get("URL"))
                        else:
                            st.warning("Video unavailable")
                        st.markdown(f"❤️ {video.get('Likes', 0):,} | 👁️ {video.get('Views', 0):,} | 💬 {video.get('Comments', 0):,}")
                        st.markdown(f"📅 {video.get('Published At Formatted', 'N/A')}")
                        desc = video.get('Description', '')
                        if desc and len(desc) > 100:
                            st.markdown(f"{desc[:100]}...")
                        else:
                            st.markdown(desc)
                        st.markdown(f"[Watch on YouTube]({video.get('URL')})")
        else:
            for video in sorted_videos:
                with st.expander(f"{video.get('Published At Formatted', 'Video')}"):
                    cols = st.columns([1, 3])
                    with cols[0]:
                        if video.get("URL") != "N/A":
                            st.video(video.get("URL"))
                        else:
                            st.warning("Video unavailable")
                    with cols[1]:
                        st.write(f"ID: {video.get('ID')}")
                        st.write(f"Title: {video.get('Title')}")
                        st.write(f"Likes: {video.get('Likes', 0):,}")
                        st.write(f"Views: {video.get('Views', 0):,}")
                        st.write(f"Comments: {video.get('Comments', 0):,}")
                        st.write(f"Published: {video.get('Published At Formatted', 'N/A')}")
                        if video.get('Tags'):
                            st.write(f"Tags: {', '.join(video.get('Tags'))}")
                        st.write(f"Category ID: {video.get('Category')}")
                        st.write("Description:")
                        st.write(video.get('Description', 'No description'))
                        st.markdown(f"[Watch on YouTube]({video.get('URL')})")

        st.download_button(
            "Download All Videos Data",
            json.dumps(video_list, indent=4),
            f"{channel_name}_videos.json",
            key=f"yt_videos_{channel_name}"
        )

###################
### MAIN APP ####
###################

def display_disclaimer():
    """Display disclaimer"""
    with st.expander("📝 Disclaimer"):
        st.markdown("""
        This tool scrapes public data from Instagram and YouTube:
        - **Instagram**: Uses web API with session authentication; requires a valid session cookie. Respect terms of service and rate limits.
        - **YouTube**: Uses Data API v3; requires a valid API key (10,000 units/day quota).
        - Only public data is accessed. Private accounts/channels show limited info.
        """)

def main():
    st.title("🔍 Social Media Scraper")
    st.markdown("Scrape public data from Instagram and YouTube")
    display_disclaimer()

    if not INSTAGRAM_COOKIE or INSTAGRAM_COOKIE == "YOUR_INSTAGRAM_SESSIONID":
        st.warning("Please set a valid Instagram session cookie in the INSTAGRAM_COOKIE variable to scrape Instagram data.")

    platform_tabs = st.tabs(["Instagram", "YouTube"])

    with platform_tabs[0]:
        with st.form("instagram_form"):
            insta_username = st.text_input("Instagram Username", placeholder="e.g. instagram", key="insta_input")
            insta_submit = st.form_submit_button("Scrape Instagram")
        if insta_submit and insta_username:
            with st.spinner("Scraping Instagram data..."):
                user_info, videos, images = scrape_instagram_user(insta_username.lstrip('@'))
                tabs = st.tabs(["Profile", "Photos", "Videos"])
                with tabs[0]:
                    display_instagram_user_info(user_info)
                with tabs[1]:
                    display_instagram_media(images, "image", insta_username)
                with tabs[2]:
                    display_instagram_media(videos, "video", insta_username)

    with platform_tabs[1]:
        with st.form("youtube_form"):
            yt_channel = st.text_input("YouTube Channel Name", placeholder="e.g. Google", key="yt_input")
            yt_num_items = st.number_input("Number of Videos to Scrape", min_value=1, max_value=50, value=5, key="yt_num_items")
            yt_submit = st.form_submit_button("Scrape YouTube")
        if yt_submit and yt_channel:
            with st.spinner("Scraping YouTube data..."):
                channel_info, videos = scrape_youtube_channel(yt_channel, yt_num_items)
                tabs = st.tabs(["Profile", "Videos"])
                with tabs[0]:
                    display_youtube_channel_info(channel_info)
                with tabs[1]:
                    display_youtube_videos(videos, channel_info.get('Title', yt_channel) if not isinstance(channel_info, str) else yt_channel)

def local_css():
    """Add custom CSS styling"""
    st.markdown("""
    <style>
    .block-container {
        padding-top: 2rem;
        padding-bottom: 2rem;
    }
    .stTabs [data-baseweb="tab-list"] {
        gap: 8px;
    }
    .stTabs [data-baseweb="tab"] {
        border-radius: 4px 4px 0px 0px;
        padding: 10px 16px;
        background-color: black;
    }
    .stTabs [aria-selected="true"] {
        background-color: #4e89ae;
        color: black;
    }
    div.stButton > button:first-child {
        width: 100%;
    }
    </style>
    """, unsafe_allow_html=True)

if __name__ == "__main__":
    local_css()
    main()
```



