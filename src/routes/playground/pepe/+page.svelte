<script>
	import { onMount, onDestroy } from 'svelte';
	import axios from 'axios';
	import { Chart } from 'frappe-charts';

	let errorMessage = '';

	let tokenName = '';
	let tokenRank = null;
	let tokenPrice = 'Loading...';
	let tokenImage = '';
	let tokenPercentageChange = null;
	let tokenVolume = null;
	let tokenCirculatingSupply = null;
	let tokenPriceChart;
	let lastUpdated = '';

	async function fetchTokenData() {
		try {
			const response = await axios.get(
				'https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&ids=pepe'
			);

			if (response.data && response.data.length > 0) {
				const tokenData = response.data[0];
				const lastUpdatedTimestamp = tokenData?.last_updated;
				localStorage.setItem(
					'tokenData',
					JSON.stringify({ data: tokenData, lastUpdated: lastUpdatedTimestamp })
				);
				populateTokenData(tokenData, lastUpdatedTimestamp);
			}
		} catch (error) {
			console.error('Error fetching token price:', error);
			const savedTokenData = localStorage.getItem('tokenData');
			if (savedTokenData !== null) {
				const parsedData = JSON.parse(savedTokenData);
				const tokenData = parsedData.data;
				const lastUpdatedTimestamp = parsedData.lastUpdated;
				populateTokenData(tokenData, lastUpdatedTimestamp);
			} else {
				tokenPrice = 'Error fetching price';
				errorMessage = 'Error fetching token data.';
			}
		}

		try {
			const sevenDaysAgo = new Date();
			sevenDaysAgo.setDate(sevenDaysAgo.getDate() - 7);
			const timestamp = Math.floor(sevenDaysAgo.getTime() / 1000);

			const response = await axios.get(
				`https://api.coingecko.com/api/v3/coins/pepe/market_chart?vs_currency=usd&days=7&interval=daily&start_timestamp=${timestamp}`
			);

			if (response.data && response.data.prices) {
				localStorage.setItem('historicalData', JSON.stringify(response.data.prices));
				createTokenPriceChart(response.data.prices);
			}
		} catch (error) {
			console.error('Error fetching historical token price data:', error);
			const historicalData = localStorage.getItem('historicalData');
			if (historicalData !== null) {
				createTokenPriceChart(JSON.parse(historicalData));
			}
		}
	}

	function populateTokenData(tokenData, lastUpdatedTimestamp) {
		tokenName = tokenData?.name || 'Error retrieving token Name';
		tokenPrice = tokenData?.current_price?.toFixed(10) || 'Price data not available';
		tokenImage = tokenData?.image;
		tokenVolume = formatNumber(tokenData?.total_volume) || 'Total Volume data not available';
		tokenPercentageChange = tokenData?.price_change_percentage_24h?.toFixed(2) || null;
		tokenRank = tokenData?.market_cap_rank || 'Error retrieving token Rank';
		tokenCirculatingSupply =
			formatNumber(tokenData?.circulating_supply) || 'Circulating supply data not available';
		if (lastUpdatedTimestamp) {
			lastUpdated = new Date(lastUpdatedTimestamp).toLocaleString();
		} else {
			lastUpdated = 'Error retrieving last updated time';
		}
	}

	function createTokenPriceChart(prices) {
		const chartData = {
			labels: prices.map((price) =>
				new Date(price[0]).toLocaleDateString('en-US', { month: 'short', day: 'numeric' })
			),
			datasets: [
				{
					name: `$${tokenName}`,
					values: prices.map((price) => price[1])
				}
			]
		};

		tokenPriceChart = new Chart('#price-chart', {
			data: chartData,
			type: 'line',
			height: 300,
			colors: ['#fff']
		});
	}

	function formatNumber(number) {
		const formatter = new Intl.NumberFormat('en-US', {
			minimumFractionDigits: 2,
			maximumFractionDigits: 2
		});

		if (number >= 1_000_000_000_000) {
			return formatter.format(number / 1_000_000_000_000) + 'T';
		} else if (number >= 1_000_000_000) {
			return formatter.format(number / 1_000_000_000) + 'B';
		} else if (number >= 1_000_000) {
			return formatter.format(number / 1_000_000) + 'M';
		} else if (number >= 1_000) {
			return formatter.format(number / 1_000) + 'K';
		} else {
			return formatter.format(number);
		}
	}

	onDestroy(() => {
		if (tokenPriceChart) {
			tokenPriceChart.destroy();
		}
	});

	onMount(async () => {
		await fetchTokenData();
	});
</script>

<div class="h-screen flex items-center justify-center">
	<div class="rounded-xl p-4 border border-black-500 shadow-lg w-128">
		<div class="flex items-center mb-8">
			<div class="flex grow">
				{#if tokenImage}
					<img class="rounded-lg w-10 h-auto" src={tokenImage} alt={tokenName} />
				{:else}<p>Image not available</p>
				{/if}
				<div class="flex flex-col ml-2">
					<h2 class="text-xl">{tokenName}</h2>
					<small class="text-black-500 leading-none">${tokenPrice}</small>
				</div>
			</div>
			<div class="flex flex-col ml-2">
				{#if tokenRank !== null}
					<h3 class="text-xl text-right">#{tokenRank}</h3>
					<small class="text-black-500 leading-none">Ranking</small>
				{:else}
					<p>Token Rank not available</p>
				{/if}
			</div>
		</div>

		<div class="flex mb-4">
			<!-- % Change -->
			<div class="flex flex-col w-1/3 mr-16">
				{#if tokenPercentageChange !== null}
					<h3 class="text-2xl">{tokenPercentageChange}%</h3>
					<small class="text-black-500 leading-none">24h change</small>
				{:else}
					<p>Percentage change not available</p>
				{/if}
			</div>
			<!-- Total Volume -->
			<div class="flex flex-col w-1/3 mr-16">
				{#if tokenVolume !== null}
					<h3 class="text-2xl">{tokenVolume}</h3>
					<small class="text-black-500 leading-none">Total Volume</small>
				{:else}
					<p>Circulating supply data not available</p>
				{/if}
			</div>
			<!-- Circulating Supply -->
			<div class="flex flex-col w-1/3">
				{#if tokenCirculatingSupply !== null}
					<h3 class="text-2xl">{tokenCirculatingSupply}</h3>
					<small class="text-black-500 leading-none">Circulating Supply</small>
				{:else}
					<p>Circulating supply data not available</p>
				{/if}
			</div>
		</div>

		<!-- Charting -->
		<div id="price-chart" />
		<div class="text-right">
			<span
				class="inline-flex items-center gap-x-1.5 rounded-md px-2 py-1 text-xs font-medium text-white ring-1 ring-inset ring-black-400"
			>
				<svg class="h-1.5 w-1.5 fill-black-400" viewBox="0 0 6 6" aria-hidden="true">
					<circle cx="3" cy="3" r="3" />
				</svg>
				{lastUpdated}
			</span>
		</div>
	</div>
</div>
