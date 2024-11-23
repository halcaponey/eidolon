<script>
	/**
	 * TODO:
	 * - try margins
	 * - previous/next day buttons
	 * - better colors
	 */
	import { Temporal } from 'temporal-polyfill';

	let now = $state(new Date());

	let isToday = $state(true);

	let data = $state(null);

	let cal = $state([]);

	let timezone = Intl.DateTimeFormat().resolvedOptions().timeZone;

	let locale = $state('default');

	let wantedDate = $state(null);

	class CycleState {
		constructor(text) {
			this.state = text;
		}

		invert() {
			return new CycleState(this.state === 'day' ? 'night' : 'day');
		}

		// duration in minutes
		get duration() {
			return this.state === 'day' ? 100 : 50;
		}

		toString() {
			return this.state;
		}
	}

	$effect(() => {
		setTimeout(() => {
			now = new Date();
		}, 1_000);
	});

	$effect(async () => {
		locale = navigator.language;

		const res = await fetch('https://api.warframestat.us/pc/cetusCycle');
		if (!res.ok) return;
		data = await res.json();

		const activation = Temporal.Instant.from(data.activation).toZonedDateTimeISO(timezone);

		wantedDate = activation.toPlainDate().toString();
	});

	$effect(() => {
		if (data == null) {
			return;
		}

		let start = Temporal.Instant.from(data.activation).toZonedDateTimeISO(timezone);

		const wantedDateObject = new Date(wantedDate);

		const wantedTemporal = Temporal.ZonedDateTime.from({
			timeZone: timezone,
			year: wantedDateObject.getFullYear(),
			month: wantedDateObject.getMonth() + 1,
			day: wantedDateObject.getDate(),
			hour: 0,
			minute: 0,
			second: 0
		});

		isToday = true;

		let isDayOk = false;
		while (!isDayOk) {
			const startOfDay = start.withPlainTime(Temporal.PlainTime.from('00:00:00'));
			const diff = start.since(startOfDay).total({ unit: 'hours' });
			/**
			 * 24 * 60 / 150 = 9.6
			 * ajouter x heure a la date actuelle
			 * 150 * 9 = 1350 apres midi
			 * 150 * 10 = 1500 avant midi
			 */
			const minuteInc = diff < 12 ? 1500 : 1350;
			const minuteDec = diff > 12 ? -1500 : -1350;

			const difference = wantedTemporal.since(startOfDay).total({ unit: 'hours' });

			if (difference >= 24) {
				start = start.add({ minutes: minuteInc });
				isToday = false;
			} else if (difference <= -24) {
				start = start.add({ minutes: minuteDec });
				isToday = false;
			} else {
				isDayOk = true;
			}
		}

		const startDay = start.day;

		const dayData = [];

		let instant = Temporal.ZonedDateTime.from(start);
		let state = new CycleState(data.state);

		while (startDay === instant.day) {
			dayData.push({ state, start: instant });
			const duration = Temporal.Duration.from({ minutes: state.duration });
			instant = instant.add(duration);
			state = state.invert();
		}

		instant = Temporal.ZonedDateTime.from(start);
		state = new CycleState(data.state);

		while (startDay === instant.day) {
			state = state.invert();
			const duration = Temporal.Duration.from({ minutes: -state.duration });
			instant = instant.add(duration);
			dayData.unshift({ state, start: instant });
		}

		cal = dayData.map(({ state, start }, i, array) => {
			let time = state.duration;

			if (i === 0) {
				const instant = array[1].start;
				const startOfDay = instant.withPlainTime(Temporal.PlainTime.from('00:00:00'));
				const difference = instant.since(startOfDay);
				time = difference.total({ unit: 'minutes' });
				start = startOfDay;
			}

			if (i === array.length - 1) {
				const instant = start;
				const startOfDay = instant.withPlainTime(Temporal.PlainTime.from('00:00:00'));
				const endOfDayInstant = startOfDay.add({ hours: 24 });
				const difference = endOfDayInstant.since(instant);
				time = difference.total({ unit: 'minutes' });
			}

			const style = `height: ${(4 * time) / 100}rem;`;
			return { state, start, style };
		});
	});

	function dateToDisplayTime(date) {
		return date.toTimeString().slice(0, 5);
	}

	function dateToHeight(date) {
		const h = Number.parseInt(date.toTimeString().slice(0, 2), 10);
		const m = Number.parseInt(date.toTimeString().slice(3, 5), 10);
		const time = h * 60 + m;
		return `${(4 * time) / 100}rem`;
	}
</script>

<input type="date" bind:value={wantedDate} />
<div class="cal">
	{#if isToday}
		<div class="now" style:top={dateToHeight(now)}>{dateToDisplayTime(now)}</div>
	{/if}
	{#each cal as c}
		<div class="item {c.state}" style={c.style}>
			<span class="state">{c.state}</span>
			{c.start.toLocaleString(locale, { hour: 'numeric', minute: 'numeric' })}
		</div>
	{/each}
</div>

<style>
	.cal {
		position: relative;
	}

	.now {
		position: absolute;
		width: 100%;
		color: green;
		border-top: 1px solid green;
	}

	.item {
		display: flex;
		font-size: 0.75rem;
		font-family: 'Courier New', Courier, monospace;
	}

	.day {
		background-color: white;
		color: black;
	}

	.night {
		background-color: black;
		color: white;
	}

	.state {
		display: block;
		width: 5rem;
	}
</style>
