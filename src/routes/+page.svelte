<script>
	import { Temporal } from 'temporal-polyfill';

	let cal = $state([]);

	let timezone = $state(Intl.DateTimeFormat().resolvedOptions().timeZone);

	let locale = $state('default');

	class CycleState {
		constructor(text) {
			this.state = text;
		}

		invert() {
			return new CycleState(this.state === 'day' ? 'night' : 'day');
		}

		// duration in minutes
		duration() {
			return this.state === 'day' ? 100 : 50;
		}

		toString() {
			return this.state;
		}
	}

	$effect(async () => {
		locale = navigator.language;

		const res = await fetch('https://api.warframestat.us/pc/cetusCycle');
		if (!res.ok) return;
		const data = await res.json();

		let start = Temporal.Instant.from(data.activation).toZonedDateTimeISO(timezone);
		let state = new CycleState(data.state);

		const startDay = start.day;

		const dayData = [];

		while (startDay === start.day) {
			dayData.push({ state, start });
			const duration = Temporal.Duration.from({ minutes: state.duration() });
			start = start.add(duration);
			state = state.invert();
		}

		start = Temporal.Instant.from(data.activation).toZonedDateTimeISO(timezone);
		state = new CycleState(data.state);

		while (startDay === start.day) {
			state = state.invert();
			const duration = Temporal.Duration.from({ minutes: -state.duration() });
			start = start.add(duration);
			dayData.unshift({ state, start });
		}

		cal = dayData.map(({ state, start }, i, array) => {
			let time = state.duration();

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
</script>

{#each cal as c}
	<div class="item {c.state}" style={c.style}>
		<span class="state">{c.state}</span>
		{c.start.toLocaleString(locale, { hour: 'numeric', minute: 'numeric' })}
	</div>
{/each}

<style>
	.item {
		display: flex;
		font-size: small;
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
