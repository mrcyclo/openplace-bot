<script setup>
    import { chain, flatMap, sumBy } from 'lodash';
    import { onMounted, onUnmounted, ref, useTemplateRef } from 'vue';
    import { SelectFile, ReadFile, Request, WriteSettings, ReadSettings, OpenURL } from '../wailsjs/go/main/App';
    import { alert, generateId, imageDataFromBuffer, input, isUnsignedInteger, numberFormat, randomstring, sleep } from './helpers';
    import { similarColor, dithering, convert, palette } from './palette';

    const canvas = useTemplateRef('canvas');
    const settings = ref({
        tileX: null,
        tileY: null,
        pX: null,
        pY: null,
        image: null,
        dithering: false,
        usePremiumColors: false,
        buyCharges: false,
        buyMaxCharges: 0,
        sleep: 60,
        requestConcurrent: 5,
        buyMissingColors: false,
        users: [],
        baseUrl: 'http://localhost',
    });
    const logs = ref([]);
    const loading = ref(false);
    const running = ref(false);
    const stopping = ref(false);
    const userTableKey = ref(Date.now());
    const baseUrl = ref(null);
    const remainingPixels = ref(0);
    const totalPixels = ref(0);

    let userTableRefreshTimer = null;

    function log(message) {
        const text = `[${new Date().toLocaleTimeString()}] ${message}`;
        logs.value.push(text);
        logs.value.splice(0, logs.value.length - 100);
    }

    function url(path) {
        if (!settings.value.baseUrl) throw new Error('Base URL is not set.');
        return settings.value.baseUrl.replace(/\/$/, '') + '/' + path.replace(/^\//, '');
    }

    async function selectImage() {
        const filename = await SelectFile([{ pattern: '*.png;*.jpg;*.jpeg;*.bmp;*.webp', name: 'Images' }]);
        if (!filename) return;

        settings.value.image = filename;

        await loadImage();
        await writeSettings();
    }

    function saveBaseUrl() {
        log(`Base URL set to ${baseUrl.value}`);
    }


    async function loadImage() {
        if (!settings.value.image) return;

        const buffer = await ReadFile(settings.value.image);
        const bytes = Uint8Array.from(atob(buffer), (c) => c.charCodeAt(0));

        let imageData = await imageDataFromBuffer(bytes);
        if (settings.value.dithering) {
            imageData = dithering(imageData, settings.value.usePremiumColors);
        } else {
            imageData = convert(imageData, settings.value.usePremiumColors);
        }

        canvas.value.width = imageData.width;
        canvas.value.height = imageData.height;

        const ctx = canvas.value.getContext('2d');
        ctx.putImageData(imageData, 0, 0);

        totalPixels.value = imageData.width * imageData.height;
        remainingPixels.value = totalPixels.value;
    }

    async function addUser() {
        let amount = await input('Enter the amount of users to add');
        if (!amount) return;
        if (!isUnsignedInteger(amount)) {
            alert('Invalid amount', undefined, 'error');
            return;
        }

        loading.value = true;

        log(`Adding ${amount} users...`);
        amount = parseInt(amount);

        const promises = [];
        for (let i = 0; i < settings.value.requestConcurrent; i++) {
            const promise = new Promise(async (resolve, reject) => {
                while (amount > 0) {
                    amount -= 1;

                    const username = `bot-` + randomstring(12);
                    const password = randomstring();

                    try {
                        const user = {
                            id: generateId(),
                            username: username,
                            password: password,
                            enabled: true,
                        };
                        await login(user);
                        await fetchMe(user);
                        settings.value.users.push(user);
                        log(`User ${username} added.`);
                    } catch (error) {
                        reject(error);
                        return;
                    }
                }

                resolve();
            });
            promises.push(promise);
        }

        try {
            await Promise.all(promises);
        } catch (error) {
            log(error);
        }

        await writeSettings();

        loading.value = false;
    }

    async function login(user) {
        const response = await Request({
            method: 'POST',
            url: url('/login'),
            data: JSON.stringify({ username: user.username, password: user.password }),
        });
        const raw = atob(response.data);

        if (response.status !== 200) {
            throw new Error(`Failed to login with status ${response.status}. Response: ${raw}`);
        }

        user.cookie = (response.headers['Set-Cookie'] || []).map((x) => x.replace(/^(\w+=.*?)(;.*?)*$/, '$1')).join(';');
    }

    async function fetchMe(user) {
        const response = await Request({ method: 'GET', url: url('/me'), cookie: user.cookie });
        const raw = atob(response.data);

        if (response.status !== 200) {
            user.me = null;
            user.lastFetch = 0;
            throw new Error(`Failed to fetch me with status ${response.status}. Response: ${raw}`);
        }

        const data = JSON.parse(raw);
        user.me = {
            banned: data.banned,
            charges: data.charges,
            droplets: data.droplets,
            extraColorsBitmap: data.extraColorsBitmap,
            flagsBitmap: data.flagsBitmap,
            id: data.id,
            name: data.name,
        };
        user.lastFetch = Date.now();
    }

    async function writeSettings() {
        try {
            await WriteSettings(JSON.stringify(settings.value));
            log('Settings saved.');
        } catch (error) {
            log('Failed to save settings: ' + error);
        }
    }

    async function readSettings() {
        try {
            const data = await ReadSettings();
            if (!data) return;

            settings.value = JSON.parse(data);
        } catch (error) {
            log('Failed to read settings: ' + error);
        }
    }

    async function start() {
        running.value = true;
        stopping.value = false;

        while (!stopping.value) {
            try {
                await loop();
            } catch (error) {
                console.error(error);
                log('Failed to loop: ' + error);
            }

            if (stopping.value) break;

            log(`Sleeping for ${settings.value.sleep} seconds...`);
            for (let i = 0; i < settings.value.sleep; i++) {
                if (stopping.value) break;
                await sleep(1000);
            }

            if (stopping.value) break;
        }

        running.value = false;
        stopping.value = false;
        log('Stopped.');
    }

    function requestStop() {
        log('Stopping...');
        stopping.value = true;
    }

    function getCharges(user) {
        if (!user.me) return 0;

        let charges = 0;
        if (user.me.charges.count > user.me.charges.max) {
            charges = Math.floor(user.me.charges.count);
        } else {
            charges = Math.floor(user.me.charges.count + (Date.now() - user.lastFetch) / user.me.charges.cooldownMs);
            if (charges > user.me.charges.max) charges = user.me.charges.max;
        }

        return Math.max(0, charges);
    }

    async function loop() {
        log('Generating pixel queue...');
        const pixelQueue = [];
        const tileMap = new Map();
        const ctx = canvas.value.getContext('2d');
        const imageData = ctx.getImageData(0, 0, canvas.value.width, canvas.value.height);
        for (let y = 0; y < canvas.value.height; y++) {
            if (stopping.value) break;

            for (let x = 0; x < canvas.value.width; x++) {
                if (stopping.value) break;

                const imageColor = {
                    r: imageData.data[y * canvas.value.width * 4 + x * 4 + 0],
                    g: imageData.data[y * canvas.value.width * 4 + x * 4 + 1],
                    b: imageData.data[y * canvas.value.width * 4 + x * 4 + 2],
                    a: imageData.data[y * canvas.value.width * 4 + x * 4 + 3],
                };
                const imagePaletteColor = similarColor(imageColor, settings.value.usePremiumColors);
                if (imagePaletteColor.idx === 0) continue;

                const coords = {
                    tx: Math.floor(parseInt(settings.value.tileX) + (parseInt(settings.value.pX) + x) / 1000),
                    ty: Math.floor(parseInt(settings.value.tileY) + (parseInt(settings.value.pY) + y) / 1000),
                    px: (parseInt(settings.value.pX) + x) % 1000,
                    py: (parseInt(settings.value.pY) + y) % 1000,
                };

                const tileKey = `${coords.tx}-${coords.ty}`;
                if (!tileMap.has(tileKey)) {
                    log(`Getting tile ${coords.tx} ${coords.ty}...`);
                    var response = await Request({ method: 'GET', url: url(`/files/s0/tiles/${coords.tx}/${coords.ty}.png`) });
                    var raw = atob(response.data);
                    if (response.status !== 200) {
                        throw new Error(`Failed to get tile with status ${response.status}. Response: ${raw}`);
                    }

                    const buffer = Uint8Array.from(raw, (c) => c.charCodeAt(0));
                    const imageData = await imageDataFromBuffer(buffer);
                    tileMap.set(tileKey, imageData);
                }

                const tileData = tileMap.get(tileKey);
                const existColor = {
                    r: tileData.data[coords.py * tileData.width * 4 + coords.px * 4 + 0],
                    g: tileData.data[coords.py * tileData.width * 4 + coords.px * 4 + 1],
                    b: tileData.data[coords.py * tileData.width * 4 + coords.px * 4 + 2],
                    a: tileData.data[coords.py * tileData.width * 4 + coords.px * 4 + 3],
                };
                const existPaletteColor = similarColor(existColor, true);
                if (existPaletteColor.idx === imagePaletteColor.idx) continue;

                pixelQueue.push({
                    tx: coords.tx,
                    ty: coords.ty,
                    px: coords.px,
                    py: coords.py,
                    colorIdx: imagePaletteColor.idx,
                });
            }
        }

        if (stopping.value) return;

        remainingPixels.value = pixelQueue.length;

        if (pixelQueue.length === 0) {
            log('Pixel queue is empty.');
            requestStop();
            return;
        }

        const colorCountMap = new Map();
        for (const pixel of pixelQueue) {
            let count = colorCountMap.get(pixel.colorIdx) || 0;
            colorCountMap.set(pixel.colorIdx, count + 1);
        }

        const lockedColors = Array.from(colorCountMap.entries())
            .filter(([idx, count]) => count > 0)
            .map(([idx, count]) => {
                return {
                    color: palette.find((c) => c.idx === idx),
                    count,
                };
            })
            .filter((item) => item.color.isPremium)
            .toSorted((a, b) => b.count - a.count)
            .map((item) => item.color);

        const promises = [];
        let userCount = settings.value.users.length;
        let painted = 0;
        for (let i = 0; i < settings.value.requestConcurrent; i++) {
            const promise = new Promise(async (resolve) => {
                while (userCount > 0 && pixelQueue.length > 0) {
                    userCount -= 1;

                    if (stopping.value) break;

                    try {
                        const userIdx = settings.value.users.length - userCount - 1;
                        const user = settings.value.users[userIdx];
                        if (!user) continue;
                        if (!user.enabled) continue;

                        if (!user.me) {
                            log(`[${user.username}] Logging in...`);
                            await login(user);
                            await fetchMe(user);
                        }

                        if (settings.value.buyCharges && user.me.charges.count < user.me.charges.max && user.me.droplets > 500) {
                            var currentCharges = getCharges(user);
                            var amount = Math.min(10, Math.floor(user.me.droplets / 500), Math.ceil((remainingPixels.value - currentCharges) / 30));
                            if (amount > 0) {
                                var response = await Request({
                                    method: 'POST',
                                    url: url('/purchase'),
                                    data: JSON.stringify({ product: { amount, id: 80 } }),
                                    cookie: user.cookie,
                                });
                                var raw = atob(response.data);

                                if (response.status !== 200) {
                                    log(`[${user.username}] Failed to purchase charges with status ${response.status}. Response: ${raw}`);
                                } else {
                                    log(`[${user.username}] Purchased charges.`);
                                    await fetchMe(user);
                                }
                            }
                        }

                        if (user.me.charges.max < settings.value.buyMaxCharges && user.me.droplets > 500) {
                            var amount = Math.floor(Math.min(10, user.me.droplets / 500));
                            var response = await Request({
                                method: 'POST',
                                url: url('/purchase'),
                                data: JSON.stringify({ product: { amount, id: 70 } }),
                                cookie: user.cookie,
                            });
                            var raw = atob(response.data);

                            if (response.status !== 200) {
                                log(`[${user.username}] Failed to purchase max charges with status ${response.status}. Response: ${raw}`);
                            } else {
                                log(`[${user.username}] Purchased max charges.`);
                                await fetchMe(user);
                            }
                        }

                        if (settings.value.buyMissingColors && user.me.droplets > 2000) {
                            let shouldBuyColor = null;

                            const fullLockedColors = lockedColors.filter((color) => settings.value.users.every((u) => !hasColor(u, color.idx)));
                            if (fullLockedColors.length > 0) {
                                shouldBuyColor = fullLockedColors[0];
                            } else {
                                const partialLockedColors = lockedColors.filter((color) => settings.value.users.some((u) => !hasColor(u, color.idx)));
                                if (partialLockedColors.length > 0) {
                                    shouldBuyColor = partialLockedColors[0];
                                }
                            }

                            if (shouldBuyColor && !hasColor(user, shouldBuyColor.idx)) {
                                var response = await Request({
                                    method: 'POST',
                                    url: url('/purchase'),
                                    data: JSON.stringify({ product: { amount: 1, id: 100, variant: shouldBuyColor.idx } }),
                                    cookie: user.cookie,
                                });
                                var raw = atob(response.data);

                                if (response.status !== 200) {
                                    log(`[${user.username}] Failed to purchase ${shouldBuyColor.name} color with status ${response.status}. Response: ${raw}`);
                                } else {
                                    log(`[${user.username}] Purchased ${shouldBuyColor.name} color.`);
                                    await fetchMe(user);
                                }
                            }
                        }

                        let charges = getCharges(user);
                        if (charges <= 0) continue;

                        const pixels = [];
                        let pixelQueueIndex = 0;
                        while (true) {
                            if (stopping.value) break;
                            if (pixelQueueIndex >= pixelQueue.length) break;
                            if (charges <= 0) break;

                            const pixel = pixelQueue[pixelQueueIndex];
                            pixelQueue[pixelQueueIndex] = undefined;
                            pixelQueueIndex += 1;

                            if (!pixel) continue;
                            if (!hasColor(user, pixel.colorIdx)) continue;

                            pixels.push(pixel);
                            charges -= 1;
                        }

                        if (pixels.length === 0) continue;

                        const pixelsByTile = chain(pixels)
                            .groupBy((pixel) => `${pixel.tx}-${pixel.ty}`)
                            .values();
                        for (const tilePixels of pixelsByTile) {
                            var response = await Request({
                                method: 'POST',
                                url: url(`/s0/pixel/${tilePixels[0].tx}/${tilePixels[0].ty}`),
                                data: JSON.stringify({
                                    colors: flatMap(tilePixels, (pixel) => pixel.colorIdx),
                                    coords: flatMap(tilePixels, (pixel) => [pixel.px, pixel.py]),
                                    t: 'skip',
                                }),
                                cookie: user.cookie,
                            });
                            var raw = atob(response.data);

                            if (response.status !== 200) {
                                log(`[${user.username}] Failed to paint with status ${response.status}. Response: ${raw}`);
                                break;
                            }

                            var data = JSON.parse(raw);
                            if (!data.painted) {
                                log(`[${user.username}] Failed to paint: ${raw}`);
                                break;
                            }

                            log(`[${user.username}] Painted: ${data.painted}.`);
                            remainingPixels.value -= data.painted;
                            painted += data.painted;
                        }

                        await fetchMe(user);
                    } catch (error) {
                        console.error(error);
                        log(error);
                        continue;
                    }
                }

                resolve();
            });
            promises.push(promise);
        }

        await Promise.all(promises);

        await writeSettings();

        if (painted === 0) {
            log(`No pixels painted.`);
            requestStop();
            return;
        }
    }

    async function toggleAllUsers() {
        const checked = settings.value.users.every((x) => x.enabled);
        for (const user of settings.value.users) {
            user.enabled = !checked;
        }
        await writeSettings();
    }

    async function doDithering() {
        await loadImage();
        await writeSettings();
    }

    async function doTogglePremiumColors() {
        await loadImage();
        await writeSettings();
    }

    function hasColor(user, index) {
        if (!user.me) return false;
        if (index < 32) return true;
        return (user.me.extraColorsBitmap & (1 << (index - 32))) !== 0;
    }

    async function openURL(url) {
        try {
            await OpenURL(url);
        } catch (error) {
            alert(error, undefined, 'error');
        }
    }

    async function readInstanceBaseUrl() {
        const buffer = await ReadFile('instance.txt');
        const raw = atob(buffer);
        const baseUrl = raw.trim();

        const url = new URL(baseUrl);
        if (url.host === 'openplace.live') {
            alert('Botting is not allowed on openplace.live.', undefined, 'error');
            throw new Error('Botting is not allowed on openplace.live.');
        }

        return baseUrl;
    }

    async function migrateUsers() {
        const filename = await SelectFile([{ pattern: 'settings.json', name: 'Settings' }]);
        if (!filename) return;

        const buffer = await ReadFile(filename);
        const raw = atob(buffer);
        const data = JSON.parse(raw);

        for (const user of data.users) {
            const existingUser = settings.value.users.find((x) => x.username === user.username);
            if (existingUser) continue;

            settings.value.users.push({
                id: user.id,
                username: user.username,
                password: user.password,
                enabled: user.enabled,
            });
        }

        await writeSettings();
    }

    onMounted(async () => {
        loading.value = true;

        try {
            settings.baseUrl = await readInstanceBaseUrl(); // <-- здесь
        } catch (error) {
            console.error(error);
            settings.baseUrl = 'http://localhost'; // <-- здесь
        } finally {
            loading.value = false;
        }

        try {
            await readSettings();
        } catch (error) {
            log('Failed to read settings: ' + error);
        }

        try {
            await loadImage();
        } catch (error) {
            log('Failed to load image: ' + error);
        }

        userTableRefreshTimer = setInterval(() => {
            userTableKey.value = Date.now();
        }, 10000);

        loading.value = false;
    });

    onUnmounted(() => {
        clearInterval(userTableRefreshTimer);
    });
</script>

<template>
    <div class="grid-container">
        <div class="grid-item">
            <div class="border rounded p-2 h-100">
                <div class="row g-2 align-items-center">
                    <div class="col-2 text-end">
                        <label for="tileX">Tile X</label>
                    </div>
                    <div class="col-4">
                        <input type="text" v-model.number="settings.tileX" id="tileX" class="form-control" placeholder="Tile X" :disabled="loading || running" @change="writeSettings" />
                    </div>
                    <div class="col-2 text-end">
                        <label for="tileY">Tile Y</label>
                    </div>
                    <div class="col-4">
                        <input type="text" v-model.number="settings.tileY" id="tileY" class="form-control" placeholder="Tile Y" :disabled="loading || running" @change="writeSettings" />
                    </div>
                    <div class="col-2 text-end">
                        <label for="pX">P X</label>
                    </div>
                    <div class="col-4">
                        <input type="text" v-model.number="settings.pX" id="pX" class="form-control" placeholder="P X" :disabled="loading || running" @change="writeSettings" />
                    </div>
                    <div class="col-2 text-end">
                        <label for="pY">P Y</label>
                    </div>
                    <div class="col-4">
                        <input type="text" v-model.number="settings.pY" id="pY" class="form-control" placeholder="P Y" :disabled="loading || running" @change="writeSettings" />
                    </div>
                    <div class="col-2 text-end align-self-start">
                        <label for="tileX">Image</label>
                    </div>
                    <div class="col-10">
                        <div class="border rounded d-flex flex-column">
                            <canvas ref="canvas"></canvas>
                            <button type="button" class="btn btn-sm btn-primary border-0 rounded-top-0" @click="selectImage" :disabled="loading || running">
                                <i class="fa-solid fa-image"></i>
                                Select image
                            </button>
                        </div>
                    </div>
                    <div class="col-2 text-end align-self-start">
                        <label for="tileX">Options</label>
                    </div>
                    <div class="col-10">
                        <div class="d-flex flex-column">
                            <div class="form-check">
                                <input class="form-check-input" type="checkbox" id="dithering" v-model="settings.dithering" :disabled="loading" @change="doDithering" />
                                <label class="form-check-label" for="dithering">Dithering</label>
                            </div>
                            <div class="form-check">
                                <input class="form-check-input" type="checkbox" id="usePremiumColors" v-model="settings.usePremiumColors" :disabled="loading" @change="doTogglePremiumColors" />
                                <label class="form-check-label" for="usePremiumColors">Use premium colors</label>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <div class="grid-item">
            <div class="border rounded p-2 h-100">
                <div class="d-flex flex-column gap-2 h-100">
                    <div class="form-check">
                        <input class="form-check-input" type="checkbox" id="buyMissingColors" v-model="settings.buyMissingColors" :disabled="loading" @change="writeSettings" />
                        <label class="form-check-label" for="buyMissingColors">Buy missing colors</label>
                    </div>
                    <div class="form-check">
                        <input class="form-check-input" type="checkbox" id="buyCharges" v-model="settings.buyCharges" :disabled="loading" @change="writeSettings" />
                        <label class="form-check-label" for="buyCharges">Buy charges</label>
                    </div>
                    <div class="form-group">
                        <label for="buyMaxCharges" class="form-label mb-0">Buy max charges to reach: {{ numberFormat(settings.buyMaxCharges) }}</label>
                        <input type="range" class="form-range" min="0" max="1000" step="10" id="buyMaxCharges" v-model.number="settings.buyMaxCharges" :disabled="loading" @change="writeSettings" />
                    </div>
                    <div class="form-group">
                        <label for="sleep" class="form-label mb-0">Sleep between loops: {{ numberFormat(settings.sleep) }} seconds</label>
                        <input type="range" class="form-range" min="0" max="1000" step="10" id="sleep" v-model.number="settings.sleep" :disabled="loading" @change="writeSettings" />
                    </div>
                    <div class="form-group">
                        <label for="requestConcurrent" class="form-label mb-0">Request concurrent: {{ numberFormat(settings.requestConcurrent) }}</label>
                        <input
                            type="range"
                            class="form-range"
                            min="1"
                            max="10"
                            step="1"
                            id="requestConcurrent"
                            v-model.number="settings.requestConcurrent"
                            :disabled="loading"
                            @change="writeSettings"
                        />
                    </div>

                    <div class="form-group mt-auto">
                        <div class="d-flex align-items-center gap-2 mb-2">
                            <i class="fa-solid fa-earth-asia"></i>
                            <div>
                                Instance: 
                                <input type="text" v-model="settings.baseUrl" class="form-control form-control-sm d-inline-block" 
                                    style="width: 300px;" :disabled="loading || running" @change="writeSettings" />
                            </div>
                        </div>

                        <button v-if="!running" type="button" class="btn btn-primary w-100" :disabled="loading || running" @click="start">
                            <i class="fa-solid fa-play"></i>
                            Start
                        </button>
                        <button v-else type="button" class="btn btn-danger w-100" :disabled="stopping" @click="requestStop">
                            <i class="fa-solid fa-stop"></i>
                            Stop
                        </button>
                    </div>
                </div>
            </div>
        </div>

        <div class="grid-item">
            <div class="border rounded p-2 h-100 overflow-y-auto">
                <div class="d-flex align-items-center justify-content-between gap-2">
                    <div class="d-flex align-items-center gap-2">
                        <button type="button" class="btn btn-sm btn-primary" @click="addUser" :disabled="loading">
                            <i class="fa-solid fa-plus"></i>
                            Add
                        </button>
                        <button type="button" class="btn btn-sm btn-secondary" @click="migrateUsers" :disabled="loading">
                            <i class="fa-solid fa-file-import"></i>
                            Migrate
                        </button>
                    </div>
                    <div class="text-end">
                        <div :key="userTableKey + '-charges'">
                            Charges: {{ numberFormat(sumBy(settings.users, (x) => getCharges(x))) }} / {{ numberFormat(sumBy(settings.users, (x) => x.me?.charges?.max || 0)) }}
                        </div>
                        <div :key="userTableKey + '-users'">Users: {{ numberFormat(settings.users.length) }}</div>
                    </div>
                </div>
                <table class="table align-middle mb-0">
                    <thead>
                        <tr>
                            <th>
                                <input type="checkbox" class="form-check-input" :disabled="loading" :checked="settings.users.every((x) => x.enabled)" @change="toggleAllUsers" />
                            </th>
                            <th>User</th>
                            <th>Charge</th>
                            <th>Droplets</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-for="user in settings.users" :key="userTableKey + '-' + user.id">
                            <td>
                                <input type="checkbox" v-model="user.enabled" class="form-check-input" :disabled="loading" />
                            </td>
                            <td>{{ user.username }}</td>
                            <td>{{ numberFormat(getCharges(user)) }}/{{ numberFormat(user.me?.charges?.max || 0) }}</td>
                            <td>{{ numberFormat(user.me?.droplets || 0) }}</td>
                        </tr>
                    </tbody>
                </table>
            </div>
        </div>

        <div class="grid-item">
            <div class="border rounded p-2 h-100 overflow-y-auto">
                <div v-if="totalPixels > 0" class="progress mb-2">
                    <div class="progress-bar text-bg-info" :style="{ width: 100 - (remainingPixels / totalPixels) * 100 + '%' }">{{ (100 - (remainingPixels / totalPixels) * 100).toFixed(2) }}%</div>
                </div>
                <div v-for="log in logs.toReversed()" :key="generateId()" class="log-item">{{ log }}</div>
            </div>
        </div>
    </div>
</template>
