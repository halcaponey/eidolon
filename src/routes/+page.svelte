<script>
	import { Temporal } from 'temporal-polyfill';

	let cal = $state([]);

	const timezone = Intl.DateTimeFormat().resolvedOptions().timeZone;

	function invert(state) {
		return state === 'day' ? 'night' : 'day';
	}

	function getDuration(state) {
		return state === 'day' ? 100 : 50;
	}

	function extractDayFromInstant(instant) {
		const zonedDateTime = instant.toZonedDateTimeISO(timezone);
		const zonedDate = zonedDateTime.toPlainDate();
		return zonedDate;
	}

	$effect(async () => {
		const res = await fetch('https://api.warframestat.us/pc/cetusCycle');
		if (!res.ok) return;
		const data = await res.json();

		let start = Temporal.Instant.from(data.activation);
		let state = data.state;

		const startDay = extractDayFromInstant(start);

		const dayData = [];

		while (extractDayFromInstant(start).equals(startDay)) {
			dayData.push({ state, start });
			const duration = Temporal.Duration.from({ minutes: getDuration(state) });
			start = start.add(duration);
			state = invert(state);
		}

		start = Temporal.Instant.from(data.activation);
		state = data.state;

		while (extractDayFromInstant(start).equals(startDay)) {
			state = invert(state);
			const duration = Temporal.Duration.from({ minutes: -getDuration(state) });
			start = start.add(duration);
			dayData.unshift({ state, start });
		}

		cal = dayData.map(({ state, start }, i, array) => {
			let time = 100;
			if (state === 'night') {
				time = 50;
			}

			if (i === 0) {
				const instant = array[1].start;
				const zonedDateTime = instant.toZonedDateTimeISO(timezone);
				const startOfDay = zonedDateTime.withPlainTime(Temporal.PlainTime.from('00:00:00'));
				const startOfDayInstant = startOfDay.toInstant();
				const difference = instant.since(startOfDayInstant);
				time = difference.total({ unit: 'minutes' });
			}

			if (i === array.length - 1) {
				const instant = start;
				const zonedDateTime = instant.toZonedDateTimeISO(timezone);
				const startOfDay = zonedDateTime.withPlainTime(Temporal.PlainTime.from('00:00:00'));
				const startOfDayInstant = startOfDay.toInstant();
				const endOfDayInstant = startOfDayInstant.add({ hours: 24 });
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
		{c.start.toZonedDateTimeISO(timezone).toString().substring(11, 16)}
	</div>
{/each}

<style>
	.item {
		display: flex;
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
