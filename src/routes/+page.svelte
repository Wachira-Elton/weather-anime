<script lang="ts">
	import { onMount } from 'svelte';

	type Weather = {
		name: string;
		country: string;
		temperature: number;
		feelsLike: number;
		wind: number;
		humidity: number;
		code: number;
		daily: {
			time: string[];
			temperature_2m_max: number[];
			temperature_2m_min: number[];
			weather_code: number[];
		};
	};

	let city = $state('Nairobi');
	let weather = $state<Weather | null>(null);
	let loading = $state(false);
	let error = $state('');
	let offline = $state(false);

	const weatherText: Record<number, string> = {
		0: 'Clear sky',
		1: 'Mostly clear',
		2: 'Partly cloudy',
		3: 'Cloudy',
		45: 'Foggy',
		48: 'Foggy',
		51: 'Light drizzle',
		53: 'Drizzle',
		55: 'Heavy drizzle',
		61: 'Light rain',
		63: 'Rain',
		65: 'Heavy rain',
		71: 'Light snow',
		73: 'Snow',
		75: 'Heavy snow',
		80: 'Rain showers',
		81: 'Strong showers',
		82: 'Storm showers',
		95: 'Thunderstorm',
		96: 'Storm with hail',
		99: 'Heavy storm'
	};

	function icon(code: number) {
		if (code === 0) return '☀️';
		if ([1, 2].includes(code)) return '🌤️';
		if ([3, 45, 48].includes(code)) return '☁️';
		if ([51, 53, 55, 61, 63, 65, 80, 81, 82].includes(code)) return '🌧️';
		if ([95, 96, 99].includes(code)) return '⛈️';
		if ([71, 73, 75].includes(code)) return '❄️';
		return '🌦️';
	}

	function label(code: number) {
		return weatherText[code] ?? 'Changing weather';
	}

	function dateLabel(date: string) {
		return new Intl.DateTimeFormat(undefined, {
			weekday: 'short',
			month: 'short',
			day: 'numeric'
		}).format(new Date(`${date}T12:00:00`));
	}

	const forecast = $derived(
		(() => {
			const daily = weather?.daily;
			return daily
				? daily.time.map((date, i) => ({
					date,
					max: Math.round(daily.temperature_2m_max[i]),
					min: Math.round(daily.temperature_2m_min[i]),
					code: daily.weather_code[i]
				}))
				: [];
		})()
	);

	async function loadWeather(search = city) {
		const query = search.trim();

		if (!query) {
			error = 'Enter a city name.';
			return;
		}

		loading = true;
		error = '';
		offline = !navigator.onLine;

		try {
			const geoResponse = await fetch(
				`https://geocoding-api.open-meteo.com/v1/search?name=${encodeURIComponent(query)}&count=1&language=en&format=json`
			);

			if (!geoResponse.ok) throw new Error('Location search failed.');

			const geo = await geoResponse.json();
			const place = geo.results?.[0];

			if (!place) throw new Error('City not found. Try another name.');

			const forecastResponse = await fetch(
				`https://api.open-meteo.com/v1/forecast?latitude=${place.latitude}&longitude=${place.longitude}&current=temperature_2m,relative_humidity_2m,apparent_temperature,weather_code,wind_speed_10m&daily=weather_code,temperature_2m_max,temperature_2m_min&timezone=auto&forecast_days=7`
			);

			if (!forecastResponse.ok) throw new Error('Weather service unavailable.');

			const result = await forecastResponse.json();

			weather = {
				name: place.name,
				country: place.country ?? '',
				temperature: Math.round(result.current.temperature_2m),
				feelsLike: Math.round(result.current.apparent_temperature),
				humidity: result.current.relative_humidity_2m,
				wind: Math.round(result.current.wind_speed_10m),
				code: result.current.weather_code,
				daily: result.daily
			};

			city = place.name;
		} catch (err) {
			error = err instanceof Error ? err.message : 'Something went wrong.';
		} finally {
			loading = false;
		}
	}

	function useMyLocation() {
		if (!navigator.geolocation) {
			error = 'Location services are not supported on this device.';
			return;
		}

		loading = true;
		error = '';

		navigator.geolocation.getCurrentPosition(
			async (position) => {
				try {
					const { latitude, longitude } = position.coords;

					const response = await fetch(
						`https://api.open-meteo.com/v1/forecast?latitude=${latitude}&longitude=${longitude}&current=temperature_2m,relative_humidity_2m,apparent_temperature,weather_code,wind_speed_10m&daily=weather_code,temperature_2m_max,temperature_2m_min&timezone=auto&forecast_days=7`
					);

					if (!response.ok) throw new Error('Could not load your weather.');

					const result = await response.json();

					weather = {
						name: 'My location',
						country: '',
						temperature: Math.round(result.current.temperature_2m),
						feelsLike: Math.round(result.current.apparent_temperature),
						humidity: result.current.relative_humidity_2m,
						wind: Math.round(result.current.wind_speed_10m),
						code: result.current.weather_code,
						daily: result.daily
					};
				} catch {
					error = 'Could not load your local weather.';
				} finally {
					loading = false;
				}
			},
			() => {
				error = 'Location permission was denied.';
				loading = false;
			},
			{ enableHighAccuracy: false, timeout: 8000 }
		);
	}

	function handleSubmit(event: SubmitEvent) {
		event.preventDefault();
		loadWeather();
	}

	onMount(() => {
		loadWeather();

		const updateOnline = () => (offline = !navigator.onLine);
		window.addEventListener('online', updateOnline);
		window.addEventListener('offline', updateOnline);

		return () => {
			window.removeEventListener('online', updateOnline);
			window.removeEventListener('offline', updateOnline);
		};
	});
</script>

<svelte:head>
	<title>Tempest — Anime Weather</title>
	<meta
		name="description"
		content="A dramatic anime-inspired weather dashboard."
	/>
	<meta name="theme-color" content="#10111c" />
</svelte:head>

<main class:loading>
	<div class="energy energy-one"></div>
	<div class="energy energy-two"></div>

	<header>
		<div>
			<p class="eyebrow">TEMPEST WEATHER</p>
			<h1>Know your <span>battlefield.</span></h1>
			<p class="subtitle">Fast weather intel with anime-level energy.</p>
		</div>

		<button class="location-button" on:click={useMyLocation}>
			◎ Use my location
		</button>
	</header>

	<form on:submit={handleSubmit} class="search">
		<label for="city">Search location</label>
		<div class="search-row">
			<input id="city" bind:value={city} placeholder="Enter a city..." autocomplete="off" />
			<button type="submit" aria-label="Search weather">Search</button>
		</div>
	</form>

	{#if offline}
		<div class="notice">You appear to be offline. Showing the last available result.</div>
	{/if}

	{#if error}
		<div class="error" role="alert">{error}</div>
	{/if}

	{#if loading}
		<section class="loading-card" aria-live="polite">
			<div class="pulse"></div>
			Loading weather intel...
		</section>
	{:else if weather}
		<section class="hero-card">
			<div class="hero-copy">
				<p class="location">{weather.name}, {weather.country}</p>
				<div class="temperature">{weather.temperature}°</div>
				<h2>{label(weather.code)}</h2>
				<p>Feels like {weather.feelsLike}°</p>
			</div>

			<div class="weather-icon" aria-hidden="true">{icon(weather.code)}</div>
			<div class="burst"></div>
		</section>

		<section class="stats" aria-label="Current conditions">
			<div><span>Humidity</span><strong>{weather.humidity}%</strong></div>
			<div><span>Wind</span><strong>{weather.wind} km/h</strong></div>
			<div><span>Signal</span><strong>Stable</strong></div>
		</section>

		<section class="forecast">
			<div class="section-title">
				<h2>7-day forecast</h2>
				<span>Training arc</span>
			</div>

			<div class="forecast-grid">
				{#each forecast as day, index}
					<article class:today={index === 0}>
						<p>{index === 0 ? 'Today' : dateLabel(day.date)}</p>
						<div class="day-icon">{icon(day.code)}</div>
						<strong>{day.max}°</strong>
						<span>{day.min}°</span>
						<small>{label(day.code)}</small>
					</article>
				{/each}
			</div>
		</section>
	{/if}

	<footer>
		<p>Original anime-inspired design · Weather data updates automatically</p>
	</footer>
</main>

<style>
	:global(*) {
		box-sizing: border-box;
	}

	:global(body) {
		margin: 0;
		background: #0d0e17;
		color: #f8f7f2;
		font-family:
			Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI",
			sans-serif;
	}

	main {
		position: relative;
		isolation: isolate;
		min-height: 100svh;
		overflow: hidden;
		padding: clamp(1rem, 4vw, 4rem);
		background:
			radial-gradient(circle at 85% 8%, #43257e 0, transparent 28rem),
			linear-gradient(135deg, #121421, #090a10);
	}

	header,
	.search,
	.hero-card,
	.stats,
	.forecast,
	footer,
	.notice,
	.error,
	.loading-card {
		position: relative;
		z-index: 1;
		max-width: 1100px;
		margin-inline: auto;
	}

	header {
		display: flex;
		align-items: end;
		justify-content: space-between;
		gap: 1rem;
	}

	.eyebrow,
	.section-title span {
		color: #ffbd4a;
		font-size: 0.72rem;
		font-weight: 800;
		letter-spacing: 0.2em;
	}

	h1 {
		max-width: 650px;
		margin: 0.4rem 0;
		font-size: clamp(2.2rem, 7vw, 5.8rem);
		line-height: 0.95;
		letter-spacing: -0.07em;
	}

	h1 span {
		color: #ffbd4a;
	}

	.subtitle,
	.hero-copy p,
	footer {
		color: #a8a8b7;
	}

	button,
	input {
		border: 0;
		border-radius: 999px;
		font: inherit;
	}

	button {
		cursor: pointer;
		font-weight: 800;
	}

	.location-button,
	.search button {
		padding: 0.8rem 1.1rem;
		background: #ffbd4a;
		color: #17100a;
	}

	.search {
		margin-top: 2rem;
	}

	.search label {
		display: block;
		margin-bottom: 0.5rem;
		color: #a8a8b7;
		font-size: 0.85rem;
	}

	.search-row {
		display: flex;
		gap: 0.6rem;
	}

	input {
		width: min(100%, 480px);
		padding: 1rem 1.2rem;
		background: #ffffff12;
		color: white;
		outline: 1px solid #ffffff18;
	}

	input:focus {
		outline: 2px solid #ffbd4a;
	}

	.hero-card {
		display: flex;
		align-items: center;
		justify-content: space-between;
		min-height: 310px;
		margin-top: 2rem;
		padding: clamp(1.5rem, 5vw, 4rem);
		overflow: hidden;
		border: 1px solid #ffffff18;
		border-radius: 2rem;
		background: linear-gradient(115deg, #ffb43d, #ed6b3d 45%, #542884);
		box-shadow: 0 2rem 5rem #0006;
	}

	.location {
		margin: 0;
		color: #fff;
		font-weight: 700;
	}

	.temperature {
		margin: 0.4rem 0;
		font-size: clamp(5rem, 16vw, 10rem);
		font-weight: 900;
		line-height: 0.85;
		letter-spacing: -0.1em;
	}

	.hero-copy h2 {
		margin: 0;
		font-size: clamp(1.2rem, 3vw, 2rem);
	}

	.weather-icon {
		position: relative;
		z-index: 1;
		font-size: clamp(5rem, 15vw, 11rem);
		filter: drop-shadow(0 1rem 1rem #0005);
		animation: float 4s ease-in-out infinite;
	}

	.burst {
		position: absolute;
		right: 10%;
		width: 23rem;
		height: 23rem;
		border: 2px solid #ffffff55;
		border-radius: 50%;
		transform: rotate(25deg) scaleY(0.35);
	}

	.stats {
		display: grid;
		grid-template-columns: repeat(3, 1fr);
		gap: 1rem;
		margin-top: 1rem;
	}

	.stats div,
	.forecast article,
	.notice,
	.error,
	.loading-card {
		padding: 1.2rem;
		border: 1px solid #ffffff12;
		border-radius: 1.2rem;
		background: #ffffff08;
		backdrop-filter: blur(12px);
	}

	.stats span,
	.stats strong {
		display: block;
	}

	.stats span {
		color: #a8a8b7;
		font-size: 0.8rem;
	}

	.stats strong {
		margin-top: 0.3rem;
		font-size: 1.2rem;
	}

	.forecast {
		margin-top: 2rem;
	}

	.section-title {
		display: flex;
		align-items: center;
		justify-content: space-between;
	}

	.section-title h2 {
		margin: 0 0 1rem;
	}

	.forecast-grid {
		display: grid;
		grid-template-columns: repeat(7, 1fr);
		gap: 0.7rem;
	}

	.forecast article {
		min-width: 0;
		text-align: center;
	}

	.forecast article.today {
		border-color: #ffbd4a;
		background: #ffbd4a18;
	}

	.forecast article p,
	.forecast article small {
		display: block;
		overflow: hidden;
		font-size: 0.75rem;
		text-overflow: ellipsis;
		white-space: nowrap;
	}

	.forecast article p,
	.forecast article small,
	.forecast article span {
		color: #a8a8b7;
	}

	.day-icon {
		margin: 0.8rem 0;
		font-size: 2rem;
	}

	.forecast strong {
		font-size: 1.2rem;
	}

	.forecast span {
		margin-left: 0.3rem;
	}

	.forecast small {
		margin-top: 0.7rem;
	}

	.notice,
	.error,
	.loading-card {
		margin-top: 1rem;
	}

	.notice {
		color: #ffdb82;
	}

	.error {
		color: #ff9c9c;
	}

	.loading-card {
		text-align: center;
		color: #ffbd4a;
	}

	.pulse {
		width: 2rem;
		height: 2rem;
		margin: 0 auto 0.7rem;
		border: 3px solid #ffbd4a;
		border-top-color: transparent;
		border-radius: 50%;
		animation: spin 0.8s linear infinite;
	}

	footer {
		max-width: 1100px;
		margin: 2rem auto 0;
		font-size: 0.75rem;
	}

	.energy {
		position: absolute;
		z-index: 0;
		width: 40rem;
		height: 5rem;
		background: #ffbd4a1c;
		filter: blur(1.5rem);
		transform: rotate(-35deg);
		animation: drift 10s ease-in-out infinite alternate;
	}

	.energy-one {
		top: 22%;
		left: -15rem;
	}

	.energy-two {
		right: -15rem;
		bottom: 15%;
		animation-delay: -4s;
	}

	@keyframes float {
		50% {
			transform: translateY(-12px) rotate(3deg);
		}
	}

	@keyframes drift {
		to {
			transform: translate(12rem, 8rem) rotate(-35deg);
		}
	}

	@keyframes spin {
		to {
			transform: rotate(360deg);
		}
	}

	@media (max-width: 700px) {
		main {
			padding: 1rem;
		}

		header {
			display: block;
		}

		.location-button {
			margin-top: 1rem;
		}

		.hero-card {
			min-height: 260px;
		}

		.weather-icon {
			position: absolute;
			right: 1rem;
			opacity: 0.8;
		}

		.stats {
			grid-template-columns: repeat(3, 1fr);
			gap: 0.5rem;
		}

		.stats div {
			padding: 0.9rem 0.6rem;
		}

		.forecast-grid {
			display: flex;
			overflow-x: auto;
			padding-bottom: 0.5rem;
			scroll-snap-type: x mandatory;
		}

		.forecast article {
			min-width: 115px;
			scroll-snap-align: start;
		}
	}

	@media (prefers-reduced-motion: reduce) {
		* {
			animation-duration: 0.01ms !important;
			scroll-behavior: auto !important;
		}
	}

	@media (horizontal-viewport-segments: 2) {
		main {
			padding-left: calc(env(viewport-segment-left 0 0) + 2rem);
			padding-right: calc(env(viewport-segment-right 1 0) + 2rem);
		}
	}
</style>
