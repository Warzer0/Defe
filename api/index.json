const { addonBuilder, getRouter } = require("stremio-addon-sdk");
const axios = require("axios");
const cheerio = require("cheerio");

const BASE_URL = "https://downloadeverythingfromeverywhere.com";

const manifest = {
  id: "community.dle.vercel",
  version: "1.0.0",
  name: "DownloadEverything",
  description: "Direct MKV/MP4 streams from downloadeverythingfromeverywhere",
  resources: ["stream"],
  types: ["movie", "series"],
  idPrefixes: ["tt", "tmdb:"]
};

const builder = new addonBuilder(manifest);

// Keyless IMDb-to-TMDb converter using Stremio Cinemeta + site search
async function resolveTmdbId(id, type) {
  if (id.startsWith("tmdb:")) return id.replace("tmdb:", "");

  const imdbId = id.split(":")[0];
  try {
    // 1. Get title & year from Stremio's official Cinemeta API (no API key needed)
    const metaRes = await axios.get(`https://v3-cinemeta.strem.io/meta/${type}/${imdbId}.json`);
    const title = metaRes.data?.meta?.name;
    const year = metaRes.data?.meta?.releaseInfo?.split("-")[0];

    if (!title) return null;

    // 2. Search the target website for that title
    const searchRes = await axios.get(`${BASE_URL}/?q=${encodeURIComponent(title)}`, {
      headers: { "User-Agent": "Mozilla/5.0 (Linux; Android 14)" }
    });
    const $ = cheerio.load(searchRes.data);
    const prefix = type === "series" ? "/s/" : "/m/";

    let matchedId = null;

    $("ul.results li a.result-card").each((_, el) => {
      const href = $(el).attr("href") || "";
      const cardName = $(el).find(".name").text().trim();

      if (href.startsWith(prefix)) {
        const extractedId = href.replace(prefix, "").split("/")[0];
        if (!matchedId) matchedId = extractedId; // Fallback to first matching type
        if (year && cardName.includes(`(${year})`)) {
          matchedId = extractedId;
          return false; // Exact match found, break loop
        }
      }
    });

    return matchedId;
  } catch (e) {
    return null;
  }
}

builder.defineStreamHandler(async ({ type, id }) => {
  const streams = [];
  try {
    let tmdbId = null;
    let season = 1;
    let episode = 1;

    if (type === "series") {
      const parts = id.split(":");
      tmdbId = await resolveTmdbId(parts[0], type);
      season = parseInt(parts[1], 10);
      episode = parseInt(parts[2], 10);
    } else {
      tmdbId = await resolveTmdbId(id, type);
    }

    if (!tmdbId) return { streams: [] };

    const pathPrefix = type === "series" ? "s" : "m";
    const response = await axios.get(`${BASE_URL}/${pathPrefix}/${tmdbId}`, {
      headers: { "User-Agent": "Mozilla/5.0 (Linux; Android 14)" }
    });

    const $ = cheerio.load(response.data);

    if (type === "series") {
      // Find the matching season block
      let targetSeason = null;
      $("section.dl-season").each((_, sec) => {
        const h3Text = $(sec).find("h3").text().toLowerCase();
        if (h3Text.includes(`season ${season}`)) {
          targetSeason = $(sec);
        }
      });

      if (targetSeason) {
        targetSeason.find("ul.dl-eps > li").each((_, epItem) => {
          const epTitle = $(epItem).find(".ep-title").text();
          const epRegex = new RegExp(`E0*${episode}\\b`, "i");

          if (epRegex.test(epTitle)) {
            $(epItem).find(".dl-box").each((_, box) => {
              const link = $(box).find("a.dl-site");
              const href = link.attr("href");
              const title = link.attr("title") || link.text().trim();
              const tags = $(box).find(".dl-tag").map((_, t) => $(t).text()).get().join(" | ");

              if (href && (href.endsWith(".mkv") || href.endsWith(".mp4"))) {
                streams.push({
                  name: `DLE\n${tags}`,
                  title: title,
                  url: href
                });
              }
            });
          }
        });
      }
    } else {
      // Movies
      $(".dl-box").each((_, box) => {
        const link = $(box).find("a.dl-site");
        const href = link.attr("href");
        const title = link.attr("title") || link.text().trim();
        const tags = $(box).find(".dl-tag").map((_, t) => $(t).text()).get().join(" | ");

        if (href && (href.endsWith(".mkv") || href.endsWith(".mp4"))) {
          streams.push({
            name: `DLE\n${tags}`,
            title: title,
            url: href
          });
        }
      });
    }
  } catch (err) {
    console.error(err.message);
  }

  return { streams };
});

const addonInterface = builder.getInterface();
const router = getRouter(addonInterface);

module.exports = (req, res) => {
  router(req, res, () => {
    res.statusCode = 404;
    res.end(JSON.stringify({ error: "Not found" }));
  });
};
