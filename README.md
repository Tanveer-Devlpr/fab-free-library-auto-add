# Fab Free Library Auto-Add (v2.7)

**A community-made Tampermonkey userscript for helping manage free listings on Fab.**

> **Unofficial project.** Not affiliated with, maintained by, or endorsed by Epic Games, Fab, or Quixel. Read Fab’s current Terms of Service and applicable product licenses before using automation. The script is shared as-is, without a guarantee that it will work on future versions of Fab.

## What it does

- Uses the **Add to My Library** control on Fab search-result cards, without intentionally opening individual listing pages.
- Restricts automatic additions to cards that visibly display **Free**; does not click Buy, Cart, or Download controls.
- Adds the **Free** search filter when needed; **preserves any publisher filter, search query, and other selected search filters**.
- Attempts to switch off **Show products in My Library** in Fab’s User preferences.
- Waits for the Free-filter URL and result grid to settle before starting.
- Scrolls search results, remembers processed listing IDs, and shows a status panel on Fab tabs.
- Limits claiming to one Fab tab at a time; includes **Run all**, **Test 5**, **Pause**, **Speed: Fast / Normal**, **Recheck Library filter**, **Inspect next**, and **Reset progress**.
- Pauses when there is a manual confirmation, unexpected page state, or apparent rate-limit/security message.

### What it does not do

It **does not** download assets, purchase paid products, bypass security checks, guarantee every listing is collected, or guarantee uninterrupted background execution. It does not make undocumented Fab API calls. When an unexpected page reload occurs, some scrolling may be required to rebuild the infinite-scroll grid.

## Get the script

**Best for copying manually:** [`Fab_Free_Auto_Add_v2_7_COPY.txt`](./Fab_Free_Auto_Add_v2_7_COPY.txt)  
**For direct installation:** [`Fab_Free_Auto_Add_v2_7.user.js`](./Fab_Free_Auto_Add_v2_7.user.js)

Both files contain **identical JavaScript**. If your browser downloads `.user.js` incorrectly, use the TXT file and paste its contents into Tampermonkey.

## Installation in Chrome

1. Install **Tampermonkey** from its official browser-extension store.
2. In Chrome, open `chrome://extensions` → **Tampermonkey → Details**, and enable **Allow user scripts** when shown. Tampermonkey may alternatively require Chrome’s Developer mode, depending on your setup.
3. Open the Tampermonkey **Dashboard** → **Create a new script**.
4. Open [`Fab_Free_Auto_Add_v2_7_COPY.txt`](./Fab_Free_Auto_Add_v2_7_COPY.txt) on GitHub, click **Raw** or copy the full contents, then paste them over the default editor content.
5. Press **Ctrl+S** to save. Ensure the new script is **Enabled**.
6. Disable other versions of the Fab auto-add script so only one version is active.
7. Open [Fab Search](https://www.fab.com/search) and sign in normally.
8. Optionally choose a publisher or a search term. Your selected publisher should stay selected.
9. Refresh Fab, locate the **Fab · Free Card Auto-Add v2.7** panel, and click **Run all**.

## A safe first run

- On a fresh installation, use **Test 5** and verify the resulting products directly in **My Library** before using **Run all**.
- **Important:** Test 5 uses the cumulative local count of additions; if you have already recorded five or more additions with older versions, use **Run all** instead. Do not reset history merely to rerun Test 5.
- The script may perform a single search-page navigation to apply Free. It then waits **at least 10 seconds after Free is visible in the URL**, and longer if the grid is still loading.
- If Fab asks you to sign in, approve a license, or complete a security check, handle that manually.
- Use **Pause** to stop. If you see a rate-limit message, do not try to work around it; resume only when Fab allows normal activity.

## Controls

| Control | What it does |
| --- | --- |
| Test 5 | Stop when the stored total reaches five confirmed additions (best for fresh installs). |
| Run all | Process visible eligible free listings and continue scrolling. |
| Pause | Stop further automated actions; an already-sent click may still complete. |
| Speed: Fast / Normal | Changes the spacing between confirmed additions. |
| Recheck Library filter | Re-evaluates the **Show products in My Library** preference. |
| Inspect next | Highlights the next unprocessed card and logs the identified action to the browser console. |
| Reset progress | Clears the script’s local history, **not** the Fab library. Use only deliberately. |

## Troubleshooting

- **Panel missing:** Enable the script and Chrome’s **Allow user scripts** setting; refresh the Fab tab.
- **Free filter recently applied:** Allow time for Fab’s URL and product grid to stabilize. Do not repeatedly reload the page.
- **User preferences not found:** Fab can change its button labels and layout. Open **User preferences** manually, disable **Show products in My Library**, save, then choose **Recheck Library filter**.
- **Script stops on a card:** Check for a changed UI, modal, or license prompt. Use **Inspect next** and the console, and avoid forcing uncertain clicks.
- **Already owned listings:** The script records recognized owned items; hiding owned items may take time to affect displayed results.
- **Multiple tabs:** Control panels may be visible on all Fab tabs, but claiming is restricted to a single runner.
- **Minimized/unfocused Chrome:** Background scheduling is controlled by Chrome. Do not assume identical performance or uninterrupted operation.
- **After an update:** Disable the older script; v2.7 retains the v2 progress key. Do **not** click Reset if you want to keep your history.

## Project notes

- No Epic login details, session cookies, tokens, or passwords are bundled in this repository.
- The script uses local browser storage for progress, filtering markers, and a tab lock. This is local to the browser profile and origin.
- It was **syntax-checked** and exercised against **local browser simulations**. This is *not* a guarantee of continued compatibility with Fab’s live site.
- The original working v2.7 source was copied without code changes for this public package.

## Documentation

- [Complete installation and usage guide](./docs/USER_GUIDE.md)

## License

Source code is published under the [MIT License](./LICENSE). This license covers the **script source code**, not any Fab products or assets. The use of Fab and any individual asset remains subject to their respective terms and licenses.