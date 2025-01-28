<script>
	import { onMount } from "svelte";
	import { get } from "svelte/store";
	import Nav from "labs/packages/global-navbar/src/Nav.svelte";
	import SideBar from "$lib/SideBar.svelte";
	import "$lib/global.css";
	import "@xterm/xterm/css/xterm.css";
	import "@fortawesome/fontawesome-free/css/all.min.css";
	import { networkInterface, startLogin } from "$lib/network.js";
	import {
		cpuActivity,
		diskActivity,
		cpuPercentage,
		diskLatency,
	} from "$lib/activities.js";
	import {
		introMessage,
		errorMessage,
		unexpectedErrorMessage,
	} from "$lib/messages.js";
	import { displayConfig } from "$lib/anthropic.js";

	export let configObj = null;
	export let processCallback = null;
	export let cacheId = null;
	export let cpuActivityEvents = [];
	export let diskLatencies = [];
	export let activityEventsInterval = 0;
	let lastMessage = "";

	let srInput = "";
	let srMessages = [];
	function stripAnsiEscapeCodes(text) {
		return text.replace(/\x1b\[[0-9;]*m/g, "");
	}

	const srSubmit = (event) => {
		console.log(`Got ${srInput}, ${event}, ${simulateTyping}`);
		for (var i = 0; i < srInput.length; i++) {
			simulateTyping(srInput.charCodeAt(i));
		}
		simulateTyping("\n".charCodeAt(0));
		srInput = "";
		event.preventDefault();
		event.stopPropagation();
	};
	let loaded = false;
	const decoder = new TextDecoder("utf-8");
	const srHandleOutput = (buf) => {
		let data = decoder.decode(buf);
		if (data.length == 1) return;
		console.log(`Output: ${data}`);
		data = stripAnsiEscapeCodes(data);
		srPushMessage(data);
		lastMessage = data;
	};

	const srPushMessage = (msg) => {
		srMessages.push(msg);
		srMessages = srMessages; // make reactive
	};

	var term = null;
	var cx = null;
	var fitAddon = null;
	var cxReadFunc = null;
	var simulateTyping = null;
	var blockCache = null;
	var processCount = 0;
	var curVT = 0;
	var lastScreenshot = null;
	var screenshotCanvas = null;
	var screenshotCtx = null;
	function printMessage(msg) {
		for (var i of msg) {
			srPushMessage(i);
		}
	}
	function expireEvents(list, curTime, limitTime) {
		while (list.length > 1) {
			if (list[1].t < limitTime) {
				list.shift();
			} else {
				break;
			}
		}
	}
	function cleanupEvents() {
		var curTime = Date.now();
		var limitTime = curTime - 10000;
		expireEvents(cpuActivityEvents, curTime, limitTime);
		computeCpuActivity(curTime, limitTime);
		if (cpuActivityEvents.length == 0) {
			clearInterval(activityEventsInterval);
			activityEventsInterval = 0;
		}
	}
	function computeCpuActivity(curTime, limitTime) {
		var totalActiveTime = 0;
		var lastActiveTime = limitTime;
		var lastWasActive = false;
		for (var i = 0; i < cpuActivityEvents.length; i++) {
			var e = cpuActivityEvents[i];
			// NOTE: The first event could be before the limit,
			//       we need at least one event to correctly mark
			//       active time when there is long time under load
			var eTime = e.t;
			if (eTime < limitTime) eTime = limitTime;
			if (e.state == "ready") {
				// Inactive state, add the time from lastActiveTime
				totalActiveTime += eTime - lastActiveTime;
				lastWasActive = false;
			} else {
				// Active state
				lastActiveTime = eTime;
				lastWasActive = true;
			}
		}
		// Add the last interval if needed
		if (lastWasActive) {
			totalActiveTime += curTime - lastActiveTime;
		}
		cpuPercentage.set(Math.ceil((totalActiveTime / 10000) * 100));
	}
	function hddCallback(state) {
		diskActivity.set(state != "ready");
	}
	function latencyCallback(latency) {
		diskLatencies.push(latency);
		if (diskLatencies.length > 30) diskLatencies.shift();
		// Average the latency over at most 30 blocks
		var total = 0;
		for (var i = 0; i < diskLatencies.length; i++)
			total += diskLatencies[i];
		var avg = total / diskLatencies.length;
		diskLatency.set(Math.ceil(avg));
	}
	function cpuCallback(state) {
		cpuActivity.set(state != "ready");
		var curTime = Date.now();
		var limitTime = curTime - 10000;
		expireEvents(cpuActivityEvents, curTime, limitTime);
		cpuActivityEvents.push({ t: curTime, state: state });
		computeCpuActivity(curTime, limitTime);
		// Start an interval timer to cleanup old samples when no further activity is received
		if (activityEventsInterval != 0) clearInterval(activityEventsInterval);
		activityEventsInterval = setInterval(cleanupEvents, 2000);
	}
	function computeXTermFontSize() {
		return parseInt(getComputedStyle(document.body).fontSize);
	}
	function setScreenSize(display) {
		var internalMult = 1.0;
		var displayWidth = display.offsetWidth;
		var displayHeight = display.offsetHeight;
		var minWidth = 1024;
		var minHeight = 768;
		if (displayWidth < minWidth) internalMult = minWidth / displayWidth;
		if (displayHeight < minHeight)
			internalMult = Math.max(internalMult, minHeight / displayHeight);
		var internalWidth = Math.floor(displayWidth * internalMult);
		var internalHeight = Math.floor(displayHeight * internalMult);
		cx.setKmsCanvas(display, internalWidth, internalHeight);
		// Compute the size to be used for AI screenshots
		var screenshotMult = 1.0;
		var maxWidth = 1024;
		var maxHeight = 768;
		if (internalWidth > maxWidth) screenshotMult = maxWidth / internalWidth;
		if (internalHeight > maxHeight)
			screenshotMult = Math.min(
				screenshotMult,
				maxHeight / internalHeight,
			);
		var screenshotWidth = Math.floor(internalWidth * screenshotMult);
		var screenshotHeight = Math.floor(internalHeight * screenshotMult);
		// Track the state of the mouse as requested by the AI, to avoid losing the position due to user movement
		displayConfig.set({
			width: screenshotWidth,
			height: screenshotHeight,
			mouseX: 0,
			mouseY: 0,
			mouseMult: internalMult * screenshotMult,
		});
	}
	var curInnerWidth = 0;
	var curInnerHeight = 0;

	async function initTerminal() {
		try {
			await initCheerpX();
		} catch (e) {
			printMessage(unexpectedErrorMessage);
			printMessage([e.toString()]);
			return;
		}
	}

	function handleProcessCreated() {
		processCount++;
		if (processCallback) processCallback(processCount);
	}
	async function initCheerpX() {
		const CheerpX = await import("@leaningtech/cheerpx");
		var blockDevice = null;
		switch (configObj.diskImageType) {
			case "cloud":
				try {
					blockDevice = await CheerpX.CloudDevice.create(
						configObj.diskImageUrl,
					);
				} catch (e) {
					// Report the failure and try again with plain HTTP
					var wssProtocol = "wss:";
					if (configObj.diskImageUrl.startsWith(wssProtocol)) {
						// WebSocket protocol failed, try agin using plain HTTP
						plausible("WS Disk failure");
						blockDevice = await CheerpX.CloudDevice.create(
							"https:" +
								configObj.diskImageUrl.substr(
									wssProtocol.length,
								),
						);
					} else {
						// No other recovery option
						throw e;
					}
				}
				break;
			case "bytes":
				blockDevice = await CheerpX.HttpBytesDevice.create(
					configObj.diskImageUrl,
				);
				break;
			case "github":
				blockDevice = await CheerpX.GitHubDevice.create(
					configObj.diskImageUrl,
				);
				break;
			default:
				throw new Error("Unrecognized device type");
		}
		blockCache = await CheerpX.IDBDevice.create(cacheId);
		var overlayDevice = await CheerpX.OverlayDevice.create(
			blockDevice,
			blockCache,
		);
		var webDevice = await CheerpX.WebDevice.create("");
		var documentsDevice = await CheerpX.WebDevice.create("documents");
		var dataDevice = await CheerpX.DataDevice.create();
		var mountPoints = [
			// The root filesystem, as an Ext2 image
			{ type: "ext2", dev: overlayDevice, path: "/" },
			// Access to files on the Web server, relative to the current page
			{ type: "dir", dev: webDevice, path: "/web" },
			// Access to read-only data coming from JavaScript
			{ type: "dir", dev: dataDevice, path: "/data" },
			// Automatically created device files
			{ type: "devs", path: "/dev" },
			// Pseudo-terminals
			{ type: "devpts", path: "/dev/pts" },
			// The Linux 'proc' filesystem which provides information about running processes
			{ type: "proc", path: "/proc" },
			// The Linux 'sysfs' filesystem which is used to enumerate emulated devices
			{ type: "sys", path: "/sys" },
			// Convenient access to sample documents in the user directory
			{
				type: "dir",
				dev: documentsDevice,
				path: "/home/user/documents",
			},
		];
		try {
			cx = await CheerpX.Linux.create({
				mounts: mountPoints,
				networkInterface: networkInterface,
			});
		} catch (e) {
			printMessage(errorMessage);
			printMessage([e.toString()]);
			return;
		}
		cx.registerCallback("cpuActivity", cpuCallback);
		cx.registerCallback("diskActivity", hddCallback);
		cx.registerCallback("diskLatency", latencyCallback);
		cx.registerCallback("processCreated", handleProcessCreated);
		simulateTyping = cx.setCustomConsole(srHandleOutput, 40, 60);
		const display = document.getElementById("display");
		if (display) {
			setScreenSize(display);
		}
		// Run the command in a loop, in case the user exits
		loaded = true;
		while (true) {
			await cx.run(configObj.cmd, configObj.args, configObj.opts);
		}
	}
	onMount(initTerminal);
	async function handleConnect() {
		const w = window.open("login.html", "_blank");
		await cx.networkLogin();
		w.location.href = await startLogin();
	}
	async function handleReset() {
		// Be robust before initialization
		if (blockCache == null) return;
		await blockCache.reset();
		location.reload();
	}
</script>

<main class="relative w-full h-full">
	<Nav />
	<div class="absolute top-10 bottom-0 left-0 right-0">
		<slot></slot>
		{#if configObj.needsDisplay}
			<div class="absolute top-0 bottom-0 left-14 right-0">
				<canvas class="w-full h-full cursor-none" id="display"></canvas>
			</div>
		{/if}
		<div role="log">
			{#each srMessages as message}
				<p>{message}</p>
			{/each}
		</div>
		<form on:submit={srSubmit}>
			<input
				disabled={!loaded}
				title={loaded ? "Console" : "Loading..."}
				type="text"
				bind:value={srInput}
			/>
		</form>
	</div>
</main>
