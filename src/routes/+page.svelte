<script>
	// @ts-nocheck ill bother with typings later :3
	import { onMount } from "svelte";

	const apiBackend = "https://atk.wav.blue"; // I was going to do EVERYTHING from the client-side but, CORS blocked all of my requests, so we have to proxy it through my server :(
	const commands = {
		"accessory": "!hat",
		"shirt": "!shirt",
		"pants": "!pants",
		"tshirt": "!tshirt",
		"face": "!face",
		"bodycolour": "!neon",
		"purge": "!removehats",
		"delay": "!wait"
	}
	let waitDelay = 4.5;
	let ignoreList = {
		shirt: false,
		pants: false,
		tshirt: false,
		accessories: false,
		face: false,
		bodycolour: false
	};
	let clearRules = {
		shirt: false,
		pants: false,
		tshirt: false,
		accessories: true,
		face: false
	};
	let shouldAutoUpdate = true;
	let userInput;
	let outputCommand = "> No commands generated yet :(";
	let accessories = [];
	let bodyColour = 0;
	let formattedAccessories = '> No user has been fetched yet! Enter in their username and hit "Fetch" to get their accessories.';
	let copyText = "Copy to clipboard";
	let shareLink = "> Share link will show here once generated!";

	function listAccessories() {
		let output = "";
		for (let i = 0; i < accessories.length; i++) {
			const accessory = accessories[i];
			console.log(accessory);
			if (ignoreList[accessory.whatIsIt]) continue; // If in ignore settings, skip

			if (i == accessories.length - 1) {
				output += `${accessory.name} (${accessory.accessoryType})`;
			} else {
				output += `${accessory.name} (${accessory.accessoryType})\n`;
			}
		}
		return output;
	}

	function createCommandChain() { // TODO: clean up :c
		let output = "";
		let howManyCommandsIn = 0;

		console.log(`Generating command chain...`);

		if (clearRules.accessories) { // If we want to remove all of our accessories at the start of the command chain
			output += `${commands.purge} | `;
			howManyCommandsIn++;
		}
		if (clearRules.shirt) { // If we want to remove our shirt at the start of the command chain
			output += `${commands.shirt} 0 | `;
			howManyCommandsIn++;
		}
		if (clearRules.tshirt) { // If we want to remove our t-shirt at the start of the command chain
			output += `${commands.tshirt} 0 | `;
			howManyCommandsIn++;
		}
		if (clearRules.pants) { // If we want to remove our pants at the start of the command chain
			output += `${commands.pants} 0 | `;
			howManyCommandsIn++;
		}

		if (!ignoreList.bodycolour) {
			output += `${commands.bodycolour} ${bodyColour} | `;
			howManyCommandsIn++;
		}

		for (let i = 0; i < accessories.length; i++) {
			const item = accessories[i];
			if (howManyCommandsIn >= 4) {
				console.log(`Adding delay of ${waitDelay} seconds to chain...`);
				output += `${commands.delay} ${waitDelay} | `;
				howManyCommandsIn = 0;
			}

			if (item.whatIsIt == "accessory") {
				if (ignoreList.accessories) continue; // If in ignore settings, skip
				output += `${commands.accessory} ${item.id} | `;
			} else if (item.whatIsIt == "shirt") {
				if (ignoreList.shirt) continue; // If in ignore settings, skip
				output += `${commands.shirt} ${item.id} | `;
			} else if (item.whatIsIt == "pants") {
				if (ignoreList.pants) continue; // If in ignore settings, skip
				output += `${commands.pants} ${item.id} | `;
			} else if (item.whatIsIt == "tshirt") {
				if (ignoreList.tshirt) continue; // If in ignore settings, skip
				output += `${commands.tshirt} ${item.id} | `;
			} else if (item.whatIsIt == "face") {
				if (ignoreList.face) continue; // If in ignore settings, skip
				output += `${commands.face} ${item.id} | `;
			} else {
				console.warn(`Unknown item type: ${item.whatIsIt}`);
				alert(`Unknown item type was found during generation: ${item.whatIsIt}\nPress OK to continue`);
				continue;
			}

			howManyCommandsIn++;
			console.log(`Added ${item.name} to the command chain!`);
		}

		if (output.endsWith(" | ")) {
			output = output.slice(0, -3);
		}

		outputCommand = output;

		console.log("Creating share link...");
		shareLink = generateShareLink();

		return output;
	}

	function updateCommandChain() {
		if (shouldAutoUpdate) {
			console.log("Updating command chain because auto update is enabled and a value was changed...");
			createCommandChain();
			return;
		}
		console.log("Not updating command chain because auto update is disabled");
	}

	/**
     * @param {any} username
     */
	async function resolveUserIdBasedOnUsername(username) {
		const response = await fetch(`${apiBackend}/resolve/${username}`).catch((err) => {
			alert(`The request to resolve the User ID has failed...\n\nNerd Stuff: ${err}`);
			throw "Unable to resolve User ID";
		});
		const data = Number(await response.text()) || null;

		if (data) {
			console.log(`Resolved ${username} to ${data}!`);
			return data;
		} else {
			alert(`Unable to resolve ${username} to a User ID, are you sure they even exist?\nRemember, use their username and not their display name.`);
			throw "User not found";
		}
	}

	async function getUserInfo() {
		console.log(`Getting user info for ${userInput}...`);
		if (isNaN(userInput)) {
			userInput = await resolveUserIdBasedOnUsername(userInput);
		}

		let response = await fetch(`${apiBackend}/getcurrentwearing/${userInput}`).catch((err) => {
			alert(`The request to get the user's current avatar has failed...\n\nNerd Stuff: ${err}`);
			throw "Unable to get user's current avatar";
		});
		let data = await response.json();
		console.log(`Got user info for ${userInput}!`);

		accessories = data.outfit;
		bodyColour = data.body.skintone;
		formattedAccessories = listAccessories();
		createCommandChain();
		return data;
	}

	function copyCommandChainToClipboard() {
		if (outputCommand == "> No commands generated yet :(") {
			alert("You haven't generated any commands yet!");
			return;
		}

		navigator.clipboard.writeText(outputCommand).then(() => {
			copyText = "Copied!";
			setTimeout(() => {
				copyText = "Copy to clipboard";
			}, 1000);
		}).catch((err) => {
			alert(`Failed to copy to clipboard...\n\nNerd Stuff: ${err}`);
		});
	}

	function sanitiseInput(text) {
		return text.replace(/([\u2700-\u27BF]|[\uE000-\uF8FF]|\uD83C[\uDC00-\uDFFF]|\uD83D[\uDC00-\uDFFF]|[\u2011-\u26FF]|\uD83E[\uDD10-\uDDFF])/g, "");
	}
	function createSaveHash(type) {
		let data;

		if (type == "settings") {
			const w = waitDelay;
			const i = ignoreList;
			const c = clearRules;
			const s = shouldAutoUpdate;
			data = JSON.stringify({
				w,
				i,
				c,
				s
			});
		} else if (type == "outfit") {
			const sanitisedAccessories = JSON.parse(sanitiseInput(JSON.stringify(accessories)));
			const u = userInput;
			const a = sanitisedAccessories;
			const b = bodyColour;
			data = JSON.stringify({
				u,
				a,
				b
			});
		} else {
			console.warn(`Unknown save hash type: ${type}`);
			return;
		}
		
		let output;

		try {
			output = btoa(data);
		} catch (err) {
			alert(`Failed to create save hash...\n\nNerd Stuff: ${err}`);
		}

		return output;
	}
	function deleteSaveCookies(type) {
		if (type == "settings") {
			document.cookie = "atk_settings=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/;";
		} else if (type == "outfit") {
			document.cookie = "atk_outfit=; expires=Thu, 01 Jan 1970 00:00:00 UTC; path=/;";
		} else {
			console.warn(`Unknown save cookie type: ${type}`);
			return;
		}
	}

	let collapsed = true;
	function toggleCollapsible() {
		if (collapsed) {
			collapsed = false;
		} else {
			collapsed = true;
		}
	}

	function generateShareLink() {
		const link = `https://avatartokronos.wav.blue/?o=${createSaveHash("outfit")}&s=${createSaveHash("settings")}`;
		console.log("Generated share link!");
		return link;
	}

	onMount(() => {
		const urlParams = new URLSearchParams(window.location.search);
		const outfitHash = urlParams.get("o");
		const settingsHash = urlParams.get("s");

		if (settingsHash) {
			console.log("Found settings hash in URL, loading settings...");
			try {
				const settingsData = JSON.parse(atob(settingsHash));
				waitDelay = settingsData.w;
				ignoreList = settingsData.i;
				clearRules = settingsData.c;
				shouldAutoUpdate = settingsData.s;
			} catch (err) {
				alert(`Failed to load attached settings from URL...\nIt is most likely corrupted!\n\nNerd Stuff: ${err}`);
			}
		}
		if (outfitHash) {
			console.log("Found outfit hash in URL, loading outfit...");
			try {
				const outfitData = JSON.parse(atob(outfitHash));
				console.log(outfitData);
				userInput = outfitData.u;
				accessories = outfitData.a;
				bodyColour = outfitData.b;
				formattedAccessories = listAccessories();
				createCommandChain();
			} catch (err) {
				alert(`Failed to load outfit from URL...\nIt is most likely corrupted!\n\nNerd Stuff: ${err}`);
			}
		}
	});
</script>

<h1 style="margin-top: 0px; margin-bottom: 0px; font-weight: bold;">Avatar to Kronos</h1>
<p>Generate chained avatar commands based on a Roblox user's current avatar</p>
<i class="gradtext" style="font-size: smaller;">Built by Waviest, view sause code on <a href="https://github.com/WaviestBalloon/AvatarToKronos">GitHub</a></i>

<br><br><br>

<span style="display: inline-flex; flex: 1; width:100%">
	<div class="block">
		<h1 class="block-header">
			<img src="wrench.svg" alt="SettingsIcon" class="header-icon">
			<code>Settings</code>
		</h1>
		
		<p>Delay per four commands: <br><input type="range" bind:value={waitDelay} on:change={updateCommandChain} min="0" max="10" step="0.1" /> <code>{waitDelay}s<br>(4-6s is the sweet spot for Kronos)</code></p>
		<p><input type="checkbox" bind:checked={clearRules.accessories} on:change={updateCommandChain} /> Clear all accessories when ran</p>
		<p><input type="checkbox" bind:checked={clearRules.shirt} on:change={updateCommandChain} /> Clear shirt when ran</p>
		<p><input type="checkbox" bind:checked={clearRules.tshirt} on:change={updateCommandChain} /> Clear t-shirt when ran</p>
		<p><input type="checkbox" bind:checked={clearRules.pants} on:change={updateCommandChain} /> Clear pants when ran</p>
		<p><input type="checkbox" bind:checked={ignoreList.shirt} on:change={updateCommandChain} /> Ignore shirt</p>
		<p><input type="checkbox" bind:checked={ignoreList.tshirt} on:change={updateCommandChain} /> Ignore t-shirt</p>
		<p><input type="checkbox" bind:checked={ignoreList.pants} on:change={updateCommandChain} /> Ignore pants</p>
		<p><input type="checkbox" bind:checked={ignoreList.accessories} on:change={updateCommandChain} /> Ignore accessories</p>
		<p><input type="checkbox" bind:checked={ignoreList.face} on:change={updateCommandChain} /> Ignore face</p>
		<p><input type="checkbox" bind:checked={ignoreList.bodycolour} on:change={updateCommandChain} /> Ignore torso body colour</p>
		<p><input type="checkbox" bind:checked={shouldAutoUpdate} on:change={updateCommandChain} /> Regenerate chain on value change</p>
		<br>
		<button class="warning" on:click={toggleCollapsible}>DANGER ZONE OPTIONS</button>
		<div class="warning-tape-background collapsible{collapsed === true ? " collapsed" : ""}">
			<h3 class="block-header">
				<img src="red-alert.svg" width=18.72px alt="SettingsIcon" class="header-icon">
				<code style="color: #ff5c5c;">Danger Zone<br><i style="font-size: small;">Nothing here can be undone!<br>NOT IMPLEMENTED YET</i></code>
			</h3>
			<center>
				<code style="color: #ff5c5c;">/// Current Editor \\\</code><br>
				<button class="warning">CLEAR ACCESSORIES</button>
				<br><code style="color: #ff5c5c;">/// Saved Data \\\</code><br>
				<button class="warning" on:click={deleteSaveCookies("settings")}>RESET SETTINGS</button>
				<button class="warning" on:click={deleteSaveCookies("outfit")}>DELETE OUTFITS</button>
			</center>
		</div>
	</div>
	<hr class="splitter">
	<div class="block">
		<h1 class="block-header">
			<img src="import.svg" alt="ImportIcon" class="header-icon">
			<code>Create</code>
		</h1>
		
		<!--<center>
			<img src="https://t0.rbxcdn.com/30DAY-AvatarHeadshot-2BB2F5470323665D06A2307C5429B0F2-Png" width="20%" alt="AvatarPreview" class="pfp">
		</center>-->

		<p>Username/ID: <input type="search" bind:value={userInput} placeholder="Roblox username or user ID here" /> <button on:click={getUserInfo}>Fetch</button></p>
		<p>Wearing Accessories: </p>
		<textarea autocomplete="off" rows="10" style="resize: none;">{formattedAccessories}</textarea>
	</div>
	<hr class="splitter">
	<div class="block">
		<h1 class="block-header">
			<img src="clipboard-copy.svg" alt="CopyResultIcon" class="header-icon">
			<code>Result</code>
		</h1>
	
		<textarea autocomplete="off" rows="4">{outputCommand}</textarea>
		<br><br>
		<button on:click={copyCommandChainToClipboard}>{copyText}</button>
		<button on:click={createCommandChain}>Generate</button>
		<br>
		<code>Share link: <textarea autocomplete="off" class="textarea-link" rows="2">{shareLink}</textarea></code>
		<!--<button on:click={createCommandChain}>Save current outfit</button><code style="color: #ff5c5c;">OVERWRITES PREVIOUSLY SAVED OUTFIT</code>-->
	</div>
</span>

<br><br><br><br>
<code style="color: #ff5c5c;">[ Disclaimer ]<br>Due to the Roblox API's CORS configuration, this website relies on a backend (that makes the requests for you) ran by me and not Roblox<br>Blame the web team for disallowing webpages to request to their API<br>I do not store any data related to your requests<br><i>Please be responsible when using other people's avatars</i></code>
<br>
