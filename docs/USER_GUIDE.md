# Fab Free Library Auto-Add v2.7 — User Guide

This is an **unofficial** community Tampermonkey script for adding eligible **free** Fab listings to **My Library** through the existing card controls. It is not affiliated with or endorsed by Epic Games, Fab, or Quixel.

## Requirements

- Google Chrome or a compatible Chromium browser with Tampermonkey installed.
- An Epic Games account signed in at [Fab](https://www.fab.com/).
- Only **one enabled version** of this Fab userscript.
- Review Fab's current rules and each asset's license before using browser automation.

## Install using the copyable TXT file

1. Open [Fab_Free_Auto_Add_v2_7_COPY.txt](../Fab_Free_Auto_Add_v2_7_COPY.txt) in this repository.
2. Use **Raw** or GitHub's copy button to select and copy the **entire** file.
3. In Chrome, click **Tampermonkey → Dashboard → Create a new script**.
4. Replace the editor's sample code with the copied text.
5. Press **Ctrl+S**, and ensure the userscript is enabled.
6. If Tampermonkey is not running, visit `chrome://extensions` → **Tampermonkey → Details** and enable **Allow user scripts** if Chrome offers that option. Some installations also require Developer mode.
7. Disable all earlier Fab Free Auto-Add / Quixel Auto-Add scripts.
8. Open [Fab Search](https://www.fab.com/search), sign in normally, and refresh the tab.

**Alternative:** Open [Fab_Free_Auto_Add_v2_7.user.js](../Fab_Free_Auto_Add_v2_7.user.js) and install it with Tampermonkey if your browser offers direct userscript installation. Both files have identical JavaScript.

## Choose what to process

On Fab's product search page, you may choose a publisher, category, search keyword, sort order, or other filters. **The script preserves filters you select.** If no publisher is selected, eligible free products from different publishers may be processed. The script adds the **Free** filter if it is missing; it does not automatically download the assets.

## First test and full run

1. Find the floating **Fab · Free Card Auto-Add v2.7** panel.
2. For a **fresh installation**, click **Test 5** and verify the items in **My Library**.
3. When satisfied, choose **Run all**. If five or more items were already recorded in older versions, **Test 5** uses the cumulative count and may immediately finish: use **Run all** instead.
4. If the Free filter is missing, the script applies it. Fab may navigate to a new URL, and the script waits **at least 10 seconds after Free is present in the URL**, potentially longer for the result grid to stabilize.
5. It attempts to open Fab's **User preferences** and disable **Show products in My Library**. If Fab's layout has changed, do this manually by opening the sliders/settings icon near **Sort by**, turning that setting **off**, and clicking **Save**. Then use **Recheck Library filter**.
6. The script uses each eligible product card's **Add to My Library** control, waits for a visible confirmation, and scrolls for more items.
7. Leave a Fab tab open while processing. Chrome can throttle or discard unfocused/minimized tabs. Running at foreground speed is **not guaranteed**.

## Panel controls

| Control | Behavior |
| --- | --- |
| **Test 5** | Target five cumulative confirmed new additions, mainly for first-time testing. |
| **Run all** | Continue through loaded eligible free products, then scroll for more. |
| **Pause** | Prevent additional automatic actions; a request already sent may still finish. |
| **Speed: Fast / Normal** | Change spacing between completed additions. |
| **Recheck Library filter** | Recheck the hidden-owned-products preference. |
| **Inspect next** | Highlight a pending card and log diagnostic details in DevTools Console. |
| **Reset progress** | Clear only local progress history, **not** assets in your Fab library. |

## Common issues

**No panel appears:** Verify Tampermonkey is enabled, user scripts are allowed in Chrome, and this script is enabled. Refresh Fab.

**A selected publisher disappears:** Confirm you installed **v2.7**, not v2.3–v2.6; disable old versions. v2.7 is intended to preserve the selected publisher.

**Free filter appears, but claiming does not start:** Check the status panel. Fab results can take time to reload. Avoid repeatedly refreshing while the script is settling the grid.

**Could not find User preferences:** Open the settings/sliders button to the right of **Sort by**, disable **Show products in My Library**, save, then click **Recheck Library filter**.

**Multiple Fab tabs:** The panel can appear in different Fab tabs, but only one is intended to run claims at once.

**Unconfirmed addition or a dialog:** Pause, verify the asset state yourself, and handle any sign-in, security, or license prompt manually. Do not attempt to bypass it.

**Rate limit or security warning:** Stop automation until Fab allows normal activity. Do not increase the rate to circumvent restrictions.

**Unexpected page reload:** The script saves progress locally, but Fab's infinite-scroll search results may still need to load again.

**My Library differs from script counts:** Counts reflect local observed confirmations, not a guarantee of ownership; verify important items directly in Fab.

## Privacy, safety, and limits

- No login credentials, passwords, authentication tokens, or account cookies are included in the published source.
- The script uses your existing signed-in browser session and saves its own progress in local browser storage.
- No purchases, checkout actions, asset downloads, or undocumented Fab claim API endpoints are part of the intended workflow.
- Manual confirmations and site policy always take priority. Browser automation may not be permitted under Fab's applicable terms.
- Site UI changes may break the script, and it cannot guarantee that every eligible asset will be discovered or successfully added.
- The source was syntax-checked and tested on local browser simulations; that does not guarantee compatibility with all live Fab sessions.

## Updating

Edit your existing Tampermonkey script and paste the newest source, save it, and disable any old copies. **Do not select Reset progress** if you want to keep the existing local history.

## License

The code is offered under this repository's [MIT License](../LICENSE). **Fab assets themselves are not licensed under MIT**; each product's own licensing terms still apply.
