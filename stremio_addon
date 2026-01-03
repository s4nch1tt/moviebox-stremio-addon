import uvicorn
import httpx
import asyncio
import logging
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from moviebox_api.core import MovieBox

# Setup Logging for debugging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("stremio-moviebox")

app = FastAPI()

# Enable CORS for Stremio to communicate with this local server
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

# Stremio Manifest definition
MANIFEST = {
    "id": "org.moviebox.addon.resolver",
    "version": "1.2.0",
    "name": "Moviebox Searcher",
    "description": "Searches Moviebox by Title and Episode name for direct links.",
    "resources": ["stream"],
    "types": ["movie", "series"],
    "catalogs": [],
    "idPrefixes": ["tt"]
}

mb = MovieBox()

async def resolve_imdb_to_name(type: str, imdb_id: str, season: int = None, episode: int = None):
    """
    Calls Cinemeta (Stremio's metadata service) to get the real Name and Episode Title.
    Example: tt10182248 -> "The Family Man", "The Family Man"
    """
    async with httpx.AsyncClient() as client:
        try:
            url = f"https://v3-cinemeta.strem.io/meta/{type}/{imdb_id}.json"
            response = await client.get(url, timeout=10.0)
            data = response.json().get("meta", {})
            
            main_name = data.get("name")
            episode_title = ""
            
            if type == "series" and "videos" in data:
                # Iterate through metadata to find the specific episode title
                videos = data.get("videos", [])
                target = next((v for v in videos if v.get("season") == season and v.get("number") == episode), None)
                if target:
                    episode_title = target.get("title", "")
            
            return main_name, episode_title
        except Exception as e:
            logger.error(f"Metadata error for {imdb_id}: {e}")
            return None, None

async def find_moviebox_streams(type: str, name: str, season: int = None, episode: int = None, ep_name: str = ""):
    """
    Search Moviebox using the text name and drill down to the specific episode.
    """
    try:
        # Search the Moviebox database using the show/movie name
        search_results = await mb.search(name)
        if not search_results:
            logger.info(f"No Moviebox results for: {name}")
            return []

        # Assume the first result is the most relevant match
        target_item = search_results[0]
        logger.info(f"Matched: {target_item.title} (ID: {target_item.id})")
        
        # Fetch full details (includes the seasons/episodes list)
        details = await mb.get_item_details(target_item)
        
        streams = []
        
        if type == "movie":
            server_details = await mb.get_server_details(details)
            for server in server_details:
                for link in server.links:
                    streams.append({
                        "name": f"Moviebox\n{link.quality}",
                        "title": f"{target_item.title}\n{link.quality} | {link.size or 'Unknown'}\nServer: {server.server_name}",
                        "url": link.url
                    })
        
        elif type == "series" and season is not None:
            # Find the requested Season
            target_season = next((s for s in details.seasons if s.number == season), None)
            if target_season:
                # Find the requested Episode
                target_episode = next((e for e in target_season.episodes if e.number == episode), None)
                if target_episode:
                    # Extract final streaming links for this episode
                    server_details = await mb.get_server_details(target_episode)
                    for server in server_details:
                        for link in server.links:
                            streams.append({
                                "name": f"MB {link.quality}",
                                "title": f"Episode: {ep_name or f'S{season}E{episode}'}\n{link.quality} | {link.size or 'N/A'}\n{server.server_name}",
                                "url": link.url
                            })
                            
        return streams

    except Exception as e:
        logger.error(f"Moviebox Extraction Error: {e}")
        return []

@app.get("/")
@app.get("/manifest.json")
async def manifest():
    return MANIFEST

@app.get("/stream/{type}/{id}.json")
async def stream_handler(type: str, id: str):
    """
    Main Stremio stream endpoint.
    Example id: tt10182248:1:1
    """
    season = None
    episode = None
    imdb_id = id
    
    if type == "series":
        parts = id.split(":")
        if len(parts) == 3:
            imdb_id, season, episode = parts[0], int(parts[1]), int(parts[2])
        else:
            return {"streams": []}

    # 1. Convert IMDb ID to real Title using Cinemeta
    main_title, ep_title = await resolve_imdb_to_name(type, imdb_id, season, episode)
    
    if not main_title:
        logger.warning(f"Could not resolve metadata for {imdb_id}")
        return {"streams": []}

    logger.info(f"Searching Moviebox for: {main_title} (S{season}E{episode})")
    
    # 2. Search Moviebox by Name and return extracted streams
    results = await find_moviebox_streams(type, main_title, season, episode, ep_title)
    
    return {"streams": results}

if __name__ == "__main__":
    # Run server on port 7860
    uvicorn.run(app, host="0.0.0.0", port=7860)
