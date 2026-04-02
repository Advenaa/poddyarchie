# Social Platform Data Collection

Research on collecting and processing data from Discord, Twitter/X, and other social platforms.

---

## [Discord Unveiled: A Comprehensive Dataset of Public Communication (2015-2024)](https://arxiv.org/abs/2502.00627)

- **Title & Authors:** Discord Unveiled: A Comprehensive Dataset of Public Communication (2015-2024) -- Yan Aquino, Pedro Bento, Arthur Buzelin, Lucas Dayrell, Samira Malaquias, Caio Santana, Victoria Estanislau, Pedro Dutenhefner, Guilherme H. G. Evangelista, Luisa G. Porfirio, Caio Souza Grossi, Pedro B. Rigueira, Virgilio Almeida, Gisele L. Pappa, Wagner Meira Jr
- **Year:** 2025
- **Core method:** Collected data from Discord's public API across 3,167 public servers listed in Discord's Discovery feature (~10% of discoverable servers). Data was organized into structured JSON files with anonymization applied for privacy. The dataset spans from Discord's launch in 2015 through end of 2024.
- **Key results:** Produced over 2.05 billion messages from 4.74 million users -- the most extensive Discord public server dataset to date. Preliminary analysis revealed significant trends in bot utilization, linguistic diversity (English dominant with Spanish, French, Portuguese), and community themes expanding well beyond gaming into social, art, music, and memes.
- **Relevance to Podders:** Directly validates the approach of scraping Discord public servers via API for large-scale data collection. The structured JSON output format and the Discovery feature as a server discovery mechanism are both applicable to Podders' Discord ingestion pipeline.

---

## [InsightEdu: Mobile Discord Bot Management and Analytics for Educators](https://arxiv.org/abs/2511.05685)

- **Title & Authors:** InsightEdu: Mobile Discord Bot Management and Analytics for Educators -- Mihail Atanasov, Santiago Berrezueta-Guzman, Stefan Wagner
- **Year:** 2025
- **Core method:** Built a system combining a Python-based Discord bot server with a SwiftUI iOS client application to manage educational Discord interactions. The bot collects surveys, feedback, and attendance data from Discord channels and surfaces analytics through a mobile-first interface.
- **Key results:** 92% of educator participants (n=20) reported enhanced efficiency managing educational interactions through the mobile interface compared to native Discord. Demonstrated that purpose-built bot + external dashboard architecture outperforms native Discord UI for structured data collection.
- **Relevance to Podders:** The bot-as-data-collector architecture (Python Discord bot + separate analytics frontend) mirrors Podders' approach. Validates the pattern of using a Discord bot to passively collect and structure channel data, then surfacing insights through a separate dashboard layer.

---

## ["I'm in the Bluesky Tonight": Insights from a Year Worth of Social Data](https://arxiv.org/abs/2404.18984)

- **Title & Authors:** "I'm in the Bluesky Tonight": Insights from a Year Worth of Social Data -- Andrea Failla, Giulio Rossetti
- **Year:** 2024
- **Core method:** Leveraged Bluesky Social's open AT Protocol to collect a near-complete dataset covering 81% of all registered accounts (4M+ users, 235M posts). Captured the full social graph including follow, comment, repost, and quote interactions, plus the output of popular feed generator algorithms with timestamped like interactions.
- **Key results:** Produced the first high-coverage dataset of a decentralized social platform, enabling analysis of content virality, diffusion patterns, and human-machine engagement. The open protocol approach achieved 81% account coverage -- far higher than typical social media scraping efforts -- demonstrating the accessibility advantage of decentralized platforms.
- **Relevance to Podders:** Shows that open-protocol platforms (like Bluesky's AT Protocol) offer dramatically better data access than proprietary APIs. Worth considering as a future Podders data source alongside Twitter/X, especially given the Twitter API restrictions documented in the next paper.

---

## [RIP Twitter API: A Eulogy to Its Vast Research Contributions](https://arxiv.org/abs/2404.07340)

- **Title & Authors:** RIP Twitter API: A Eulogy to Its Vast Research Contributions -- Ryan Murtfeldt, Sejin Paik, Naomi Alterman, Ihsan Kahveci, Jevin D. West
- **Year:** 2024
- **Core method:** Systematic bibliometric analysis across eight academic databases cataloguing all research that used Twitter data from 2006 to 2024. Categorized 33,306 studies by discipline, citation count, publication venue, and research topic to quantify the impact of Twitter's API shutdown.
- **Key results:** Twitter data powered 33,306 studies across 8,914 venues with 610,738 total citations spanning 16 disciplines. After Twitter set the Enterprise API price at $42,000/month in 2023, published studies dropped 13% -- and the decline is expected to steepen since much 2024 research used pre-shutdown data. Major research topics lost include information dissemination, event detection, crisis response, and political behavior analysis.
- **Relevance to Podders:** Critically important context for Podders' Twitter/X scraping strategy. The $42K/month API cost makes official access impractical, reinforcing the need for alternative collection methods (scraping, third-party APIs) and diversification to other platforms like Discord and RSS feeds.

---

## Summary

These four papers collectively map the landscape of social platform data collection as it stands today. Discord's public API remains accessible and can yield billion-scale datasets when systematically collected via Discovery-listed servers (Aquino et al. 2025), while purpose-built bots provide an effective collection and analytics architecture (Atanasov et al. 2025). Meanwhile, Twitter/X data access has collapsed -- 33,000+ studies and 610,000+ citations show what was possible when the API was open, but the $42K/month paywall has already caused a 13% drop in research output (Murtfeldt et al. 2024). Decentralized platforms like Bluesky offer a promising alternative with dramatically better coverage through open protocols (Failla & Rossetti 2024). For Podders, the key takeaways are: (1) Discord API collection at scale is proven and well-documented, (2) bot-based collection with separate dashboard analytics is a validated architecture, (3) Twitter/X requires non-API collection strategies, and (4) open-protocol platforms like Bluesky should be evaluated as supplementary data sources.
