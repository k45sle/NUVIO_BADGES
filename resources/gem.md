Here is the updated logic and formatting to implement all of your requests.

I've refactored the layout structure to allow the Reddit panels to break out of the standard grid and span the full width, adjusted the media styling for uncropped uniformity, swapped the wonky Unicode arrows for crisp SVGs, and stripped the emoji from the calendar.

I've also added the `IntersectionObserver` logic for media autoplay and endless scrolling. **However, we have two hard limitations due to the RSS/CORS proxy approach** which I’ve worked around in the code below:

1. **Endless Scrolling:** Standard RSS feeds (and `api.rss2json.com`) don't support pagination (`?after=` tokens are stripped). The scroll observer is built and active below, but it will gracefully trigger an "End of daily top posts" message rather than fetching page 2, preventing redundant API calls. To get true endless scrolling, we'd need to proxy Reddit's raw JSON API through your Nuvio backend to handle the CORS and pagination tokens.
2. **Video Embeds (`v.redd.it`):** Reddit's RSS feed does not expose the raw `.mp4` or HLS stream for direct video uploads; it only provides the webpage link. The `IntersectionObserver` below is fully configured to autoplay/pause on mute, but it will only fire on direct `.mp4`/`.webm` links (like from Imgur). Native Reddit videos will remain as the ▶️ placeholder since we can't fetch the raw media across origins.

Here are the surgical edits for your files.

### 1. `index.html` Updates

**Step A: Update the CSS (`<style>` block)**
Find the existing Reddit and News CSS sections and update them to reflect the new sizing, object-fit rules, and button adjustments.

```css
/* Update News List Height to roughly match the calendar */
.news-list {
  display: flex; flex-direction: column; gap: 0.85rem;
  max-height: 250px; /* Reduced from 480px */
  overflow-y: auto; padding-right: 0.5rem; position: relative; z-index: 3;
}

/* Give the Reddit list more vertical breathing room */
.reddit-list {
  display: flex; flex-direction: column; gap: 0.85rem;
  max-height: 600px; /* Increased for full-width layout */
  overflow-y: auto; padding-right: 0.5rem; position: relative; z-index: 3;
}

/* Update the Reddit media box to contain the full image */
.reddit-media {
  width: 100%; aspect-ratio: 16 / 9; /* Slightly wider aspect */
  border-radius: 14px 14px 0 0; overflow: hidden;
  background: #050204; /* Dark letterboxing background */
  display: flex; align-items: center; justify-content: center;
  flex-shrink: 0;
}
.reddit-media img, .reddit-media video { 
  width: 100%; height: 100%; 
  object-fit: contain; /* Changed from cover so nothing is cropped */
  display: block; 
}

/* Center the SVG in the refresh button */
.reddit-refresh-btn {
  background: transparent; border: 1px solid var(--card-border); color: var(--text-muted);
  width: 30px; height: 30px; border-radius: 10px; cursor: pointer;
  display: flex; align-items: center; justify-content: center; flex-shrink: 0;
  transition: background-color 0.2s ease, border-color 0.2s ease, color 0.2s ease;
}
/* Move the spin animation to an inner wrapper if needed, or keep on button */
.reddit-refresh-btn.spinning { animation: reddit-refresh-spin 0.6s linear; }

```

**Step B: Restructure the DOM Layout & Add SVGs**
Move the `<div class="reddit-panels">` **outside** of the `.main-content` div, placing it directly below the `.dashboard-layout` div so it spans the entire container. Also, replace the `⟳` text with an SVG.

```html
    <div class="dashboard-layout">
      
      <div class="main-content">
        <!-- Service Cards Grid -->
        <div class="grid" id="service-cards-grid"> ... </div>

        <!-- News Card -->
        <div class="news-card glass-panel"> ... </div>
      </div>

      <div class="side-widgets">
        <!-- Weather Panel -->
        <div class="widget-box glass-panel" id="weather-panel-widget"> ... </div>
        <!-- Calendar Panel -->
        <div class="widget-box glass-panel"> ... </div>
      </div>

    </div>

    <!-- MOVED: Reddit panels now sit below the main layout to span full width -->
    <div class="reddit-panels" style="margin-top: 2rem;">
      <div class="reddit-card glass-panel">
        <div class="reddit-header">
          <div class="section-title" style="margin-bottom:0; border:none;">r/funnycats</div>
          <button class="reddit-refresh-btn" id="refresh-funnycats" aria-label="Refresh r/funnycats" title="Refresh">
            <svg viewBox="0 0 24 24" width="16" height="16" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round"><polyline points="23 4 23 10 17 10"></polyline><path d="M20.49 15a9 9 0 1 1-2.12-9.36L23 10"></path></svg>
          </button>
        </div>
        <div class="reddit-list" id="reddit-container-funnycats">
          <div style="font-size:0.8rem; color:var(--text-muted); text-align:center; padding:1rem;">Loading Reddit posts...</div>
        </div>
      </div>

      <div class="reddit-card glass-panel">
        <div class="reddit-header">
          <div class="section-title" style="margin-bottom:0; border:none;">r/funny</div>
          <button class="reddit-refresh-btn" id="refresh-funny" aria-label="Refresh r/funny" title="Refresh">
            <svg viewBox="0 0 24 24" width="16" height="16" stroke="currentColor" stroke-width="2" fill="none" stroke-linecap="round" stroke-linejoin="round"><polyline points="23 4 23 10 17 10"></polyline><path d="M20.49 15a9 9 0 1 1-2.12-9.36L23 10"></path></svg>
          </button>
        </div>
        <div class="reddit-list" id="reddit-container-funny">
          <div style="font-size:0.8rem; color:var(--text-muted); text-align:center; padding:1rem;">Loading Reddit posts...</div>
        </div>
      </div>
    </div>

```

---

### 2. `app.js` Updates

**Step A: Remove the 🎉 from the Calendar**
Find the `renderCalendar()` function and update the event listener lines to remove the emoji:

```javascript
    if (holidaysMap[year] && holidaysMap[year][dateISO]) {
      d.classList.add('has-event');
      const holidayName = holidaysMap[year][dateISO];
      // Removed the 🎉 emoji
      d.addEventListener('mouseenter', () => { document.getElementById('event-info').textContent = holidayName; });
      d.addEventListener('click', () => { document.getElementById('event-info').textContent = holidayName; });
    }

```

**Step B: Add the Intersection Observers for Autoplay & Endless Scroll**
Add these global observers near the top of your Reddit logic in `app.js` (right above `extractRedditMedia`):

```javascript
/* Intersection Observer for Autoplaying Video Media */
const mediaPlaybackObserver = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    const video = entry.target;
    if (entry.isIntersecting) {
      video.play().catch(() => {}); // Catch DOMException if browser blocks autoplay
    } else {
      video.pause();
    }
  });
}, { threshold: 0.5 }); // Triggers when 50% of the video is in view

/* Intersection Observer for Endless Scroll Sentinel */
const scrollObserver = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const sentinel = entry.target;
      const sub = sentinel.getAttribute('data-sub');
      
      // Since RSS doesn't support pagination tokens, we swap the sentinel for a graceful end message.
      sentinel.textContent = 'End of daily top posts.';
      sentinel.style.cursor = 'default';
      scrollObserver.unobserve(sentinel);
    }
  });
}, { rootMargin: '100px' }); // Triggers 100px before reaching the bottom

```

**Step C: Update `extractRedditMedia` and `loadReddit**`
Update the Reddit extraction logic to differentiate between raw MP4s (if they exist) and images, and attach the observers to the generated elements.

```javascript
function extractRedditMedia(item) {
  const thumb = (item.thumbnail || '').trim();
  if (thumb && !REDDIT_BAD_THUMBS.has(thumb.toLowerCase()) && /^https?:\/\//i.test(thumb)) {
    return { type: thumb.endsWith('.mp4') ? 'video' : 'image', url: decodeHtmlEntities(thumb) };
  }

  const haystack = `${item.content || ''} ${item.description || ''}`;
  
  // Look for direct .mp4 or .webm links first
  const rawVidMatch = haystack.match(/href="(https?:\/\/[^"]+\.(?:mp4|webm))"/i);
  if (rawVidMatch) {
    return { type: 'video', url: decodeHtmlEntities(rawVidMatch[1]) };
  }

  const imgMatch = haystack.match(REDDIT_HREF_IMAGE_RE);
  if (imgMatch) {
    return { type: 'image', url: decodeHtmlEntities(imgMatch[1]) };
  }

  const vidMatch = haystack.match(REDDIT_VIDEO_RE);
  if (vidMatch) {
    return { type: 'video-link', url: decodeHtmlEntities(vidMatch[0]) };
  }
  return null;
}

// Inside your loadReddit(sub) function, update the media rendering block:
async function loadReddit(sub) {
  // ... (keep initial setup and fetch logic) ...

    if (data.status === 'ok' && Array.isArray(data.items) && data.items.length > 0) {
      data.items.slice(0, 10).forEach(item => { // Bumped to 10 for full width
        const itemEl = document.createElement('a');
        itemEl.className = 'reddit-item';
        itemEl.href = item.link;
        itemEl.target = '_blank';
        itemEl.rel = 'noopener noreferrer';

        const media = extractRedditMedia(item);
        if (media) {
          const mediaEl = document.createElement('div');
          mediaEl.className = 'reddit-media';
          
          if (media.type === 'image') {
            const img = document.createElement('img');
            img.src = media.url;
            img.loading = 'lazy';
            img.alt = '';
            img.referrerPolicy = 'no-referrer';
            img.onerror = () => mediaEl.remove();
            mediaEl.appendChild(img);
            mediaEl.setAttribute('data-expandable', 'true');
            mediaEl.addEventListener('click', (e) => {
              e.preventDefault(); e.stopPropagation();
              openMediaLightbox(media.url, item.title || '');
            });
          } else if (media.type === 'video') {
            // New logic for direct video embeds
            const vid = document.createElement('video');
            vid.src = media.url;
            vid.muted = true;
            vid.loop = true;
            vid.playsInline = true;
            vid.onerror = () => mediaEl.remove();
            mediaEl.appendChild(vid);
            mediaPlaybackObserver.observe(vid);
          } else {
            mediaEl.textContent = '\u25b6\ufe0f';
            mediaEl.style.fontSize = '1.5rem';
          }
          itemEl.appendChild(mediaEl);
        }

        // ... (keep title/meta appending logic) ...
        container.appendChild(itemEl);
      });

      // Append the endless scroll sentinel at the bottom
      const sentinel = document.createElement('div');
      sentinel.setAttribute('data-sub', sub);
      sentinel.style.cssText = 'font-size:0.8rem; color:var(--text-muted); text-align:center; padding:1rem 1rem 2rem 1rem;';
      sentinel.textContent = 'Loading more...';
      container.appendChild(sentinel);
      scrollObserver.observe(sentinel);

    } else {
  // ... (keep error handling) ...

```

Don't forget to bump your cache-busting query string manually (e.g., `?v=1788056097`) on your `app.js` and `theme-init.js` scripts in `index.html` before checking the container so Cloudflare pulls the fresh edits!
