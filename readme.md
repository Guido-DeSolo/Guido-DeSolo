I’m **Guyyatsu** — a Linux enthusiast, programmer, self-hoster, electronic musician, and builder of things that probably could have been solved with an existing service but are 
much more interesting when built from scratch.

Most of my work lives somewhere at the intersection of **Linux, Python, networking, automation, audio, hardware, experimental software**, before it’s actually usable. I tend to 
self-host things because running the infrastructure myself forces me to understand what is actually happening beneath the abstraction.

My projects range from relatively conventional web applications and automation scripts to home infrastructure, internet services, music production tools, computer vision 
experiments, and systems that connect software to physical or creative processes.

I enjoy understanding the complete path through a system—whether it’s writing the application but not necessarily knowing how it will be served or proxied, debugging those 
boundaries before they become rigid, or building a system where sound and video can meaningfully interact while remaining playable and controllable.

## Audio, Synthesis & Experimental Media

Music and audio engineering are another major part of my technical interest. I produce electronic music and work with a mixture of hardware synthesizers, software synthesizers, 
modular environments, DAWs, MIDI equipment, and Linux audio infrastructure.

My setup and experiments have involved tools such as **VCV Rack**, **Ardour**, **Mixxx**, **FFmpeg**, **JACK**, **Icecast**, **hardware synthesizers**, **MIDI controllers**, and 
custom software.

This has naturally led to programming projects—I’ve built or experimented with systems for things like audio analysis, BPM/musical-key detection, metadata management, stem 
separation, sample processing, streaming, and automated organization of music libraries.

## Internet Radio

<!-- BlackIce Radio Widget -->
<div id="radio-widget">
  <div class="radio-art-wrap">
    <img
      id="radio-art"
      src="/assets/img/radio-placeholder.png"
      alt="Current album artwork"
    >
  </div>

  <div class="radio-info">
    <div class="radio-status">
      <span id="radio-status-dot"></span>
      <span id="radio-status-text">Checking broadcast...</span>
    </div>

    <div id="radio-title">BlackIce Radio</div>
    <div id="radio-artist">Waiting for stream metadata...</div>

    <audio
      id="radio-player"
      controls
      preload="none"
      src="https://radio.guyyatsu.me/live">
    </audio>
  </div>
</div>

<style>
#radio-widget {
  display: flex;
  align-items: center;
  gap: 14px;
  max-width: 520px;
  padding: 12px;
  border: 1px solid #555;
  background: #111;
  color: #ddd;
  font-family: monospace;
}

.radio-art-wrap {
  flex: 0 0 96px;
  width: 96px;
  height: 96px;
}

#radio-art {
  width: 96px;
  height: 96px;
  display: block;
  object-fit: cover;
  background: #222;
  border: 1px solid #444;
}

.radio-info {
  flex: 1;
  min-width: 0;
}

.radio-status {
  display: flex;
  align-items: center;
  gap: 7px;
  margin-bottom: 8px;
  font-size: 0.8rem;
}

#radio-status-dot {
  display: inline-block;
  width: 9px;
  height: 9px;
  border-radius: 50%;
  background: #777;
}

#radio-status-dot.online {
  background: #00d000;
  box-shadow: 0 0 5px #00d000;
}

#radio-status-dot.offline {
  background: #d00000;
  box-shadow: 0 0 5px #d00000;
}

#radio-title {
  font-size: 1rem;
  font-weight: bold;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

#radio-artist {
  margin-top: 3px;
  color: #999;
  font-size: 0.85rem;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

#radio-player {
  width: 100%;
  height: 32px;
  margin-top: 10px;
}

@media (max-width: 420px) {
  #radio-widget {
    align-items: flex-start;
  }

  .radio-art-wrap,
  #radio-art {
    width: 72px;
    height: 72px;
  }

  .radio-art-wrap {
    flex-basis: 72px;
  }
}
</style>

<script>
(() => {
  const ICECAST_STATUS =
    "https://radio.guyyatsu.me/status-json.xsl";

  const STREAM_MOUNT = "/live";

  const PLACEHOLDER_ART =
    "/assets/img/radio-placeholder.png";

  const POLL_INTERVAL = 15000;

  const art = document.getElementById("radio-art");
  const titleEl = document.getElementById("radio-title");
  const artistEl = document.getElementById("radio-artist");
  const statusDot = document.getElementById("radio-status-dot");
  const statusText = document.getElementById("radio-status-text");

  let lastTrack = null;

  function setOnline(online) {
    statusDot.classList.remove("online", "offline");

    if (online) {
      statusDot.classList.add("online");
      statusText.textContent = "ON AIR";
    } else {
      statusDot.classList.add("offline");
      statusText.textContent = "OFFLINE";
    }
  }

  function findLiveSource(data) {
    let sources = data?.icestats?.source;

    if (!sources) {
      return null;
    }

    if (!Array.isArray(sources)) {
      sources = [sources];
    }

    return sources.find(source => {
      const listenUrl = source.listenurl || "";

      try {
        return new URL(listenUrl).pathname === STREAM_MOUNT;
      } catch {
        return listenUrl.endsWith(STREAM_MOUNT);
      }
    }) || null;
  }

  function parseTrack(rawTitle) {
    if (!rawTitle) {
      return {
        artist: "",
        title: ""
      };
    }

    const separator = rawTitle.indexOf(" - ");

    if (separator === -1) {
      return {
        artist: "",
        title: rawTitle.trim()
      };
    }

    return {
      artist: rawTitle.slice(0, separator).trim(),
      title: rawTitle.slice(separator + 3).trim()
    };
  }

  async function getArtwork(artist, title) {
    if (!artist && !title) {
      return PLACEHOLDER_ART;
    }

    const term = encodeURIComponent(
      [artist, title].filter(Boolean).join(" ")
    );

    try {
      const response = await fetch(
        `https://itunes.apple.com/search` +
        `?term=${term}` +
        `&entity=song` +
        `&limit=1`
      );

      if (!response.ok) {
        throw new Error("Artwork lookup failed");
      }

      const data = await response.json();
      const result = data.results?.[0];

      if (!result?.artworkUrl100) {
        return PLACEHOLDER_ART;
      }

      /*
       * Apple normally returns a 100x100 image.
       * Request a larger version for better quality.
       */
      return result.artworkUrl100.replace(
        "100x100bb",
        "500x500bb"
      );
    } catch (error) {
      console.warn("Artwork lookup error:", error);
      return PLACEHOLDER_ART;
    }
  }

  async function updateWidget() {
    try {
      const response = await fetch(
        ICECAST_STATUS + "?_=" + Date.now(),
        {
          cache: "no-store"
        }
      );

      if (!response.ok) {
        throw new Error(
          `Icecast returned HTTP ${response.status}`
        );
      }

      const data = await response.json();
      const source = findLiveSource(data);

      if (!source) {
        setOnline(false);

        titleEl.textContent = "BlackIce Radio";
        artistEl.textContent = "Broadcast offline";
        art.src = PLACEHOLDER_ART;

        lastTrack = null;
        return;
      }

      setOnline(true);

      /*
       * Icecast usually exposes metadata through `title`.
       *
       * Mixxx normally sends:
       *
       *     Artist - Track Title
       */
      const rawTitle =
        source.title ||
        source.server_name ||
        "";

      const track = parseTrack(rawTitle);

      if (track.title) {
        titleEl.textContent = track.title;
      } else {
        titleEl.textContent = "BlackIce Radio";
      }

      if (track.artist) {
        artistEl.textContent = track.artist;
      } else {
        artistEl.textContent = "Live Broadcast";
      }

      const trackKey =
        `${track.artist}|${track.title}`;

      /*
       * Don't hit the artwork service every 15 seconds if
       * the song hasn't changed.
       */
      if (trackKey !== lastTrack) {
        lastTrack = trackKey;

        art.src = PLACEHOLDER_ART;

        const artwork = await getArtwork(
          track.artist,
          track.title
        );

        /*
         * Make sure the song didn't change while the
         * artwork request was running.
         */
        if (lastTrack === trackKey) {
          art.src = artwork;
        }
      }

    } catch (error) {
      console.warn("Radio status error:", error);

      setOnline(false);

      titleEl.textContent = "BlackIce Radio";
      artistEl.textContent = "Broadcast unavailable";
      art.src = PLACEHOLDER_ART;

      lastTrack = null;
    }
  }

  art.addEventListener("error", () => {
    if (!art.src.endsWith(PLACEHOLDER_ART)) {
      art.src = PLACEHOLDER_ART;
    }
  });

  updateWidget();

  setInterval(
    updateWidget,
    POLL_INTERVAL
  );
})();
</script>

I also operate my own internet radio infrastructure.

This combines several of my interests: music, Linux, networking, web development, streaming media, and automation.

The system uses **Icecast** for streaming, with tools like **Mixxx**, **FFmpeg**, **Nginx**, and custom web components forming the surrounding infrastructure. I’ve worked on 
integrating stream status information into my website, along with experiments involving synchronized or audio-reactive visual components.

Projects like this are representative of how I like to work: start with an objective and learn whatever is required to accomplish it.

## Hardware

I’m also comfortable working outside the purely software side of computing.

I enjoy repurposing older hardware, building unusual peripherals, experimenting with cameras and capture devices, and finding ways to make inexpensive hardware perform tasks that 
weren’t necessarily designed for.

A 3D printer is another useful tool in this toolbox—being able to design a physical component when an off-the-shelf solution doesn’t quite do what I want makes the boundary between 
software and hardware much less rigid.

These are usually the fun parts of projects.

## How I Work

I’m largely project-driven, but my approach is more about understanding systems rather than just knowing syntax. Instead of learning a technology in isolation, I usually start with 
an objective and learn whatever is required to accomplish it. This means I tend to be comfortable entering unfamiliar territory, reading documentation, experimenting, breaking 
things, inspecting logs, and working backward from failures until I understand what’s happening beneath the abstraction.

That approach has made troubleshooting one of my strongest skills—I enjoy problems where the individual components seem to work correctly but the complete system doesn’t. Those 
problems require understanding interfaces and assumptions rather than simply knowing syntax.

## Elsewhere

More of my projects, experiments, writing, and assorted things I’ve built around it can be found on my personal website:

**[guyyatsu.me](https://guyyatsu.me)**

The code is just the tip of the iceberg—the website shows more of the infrastructure, projects, and assorted things I’ve built around it.

If something here looks unusual, overengineered, experimental, or like it started with the sentence *"I wonder if I could...*" , there’s a good chance that’s exactly what happened.