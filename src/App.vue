<script setup>
import { onMounted, onUnmounted } from 'vue';
import { init, dispose, registerIndicator } from 'klinecharts';

// ROI = relative change of oi
// RC = relative change of price and volume
// VP = relative ratio of vol and price for fut & spot
// VsBtc = relative ratio of vol and price for fut & spot of market vs btc

registerIndicator({
  name: 'ROI',
  figures: [{ key: 'ro', title: 'OI ', type: 'line' }],
  calc: (data) => data,
});

registerIndicator({
  name: 'RC',
  figures: [
    { key: 'rp', title: 'Price ', type: 'line' },
    { key: 'rv', title: 'Vol ', type: 'line' },
  ],
  calc: (data) => data,
});

registerIndicator({
  name: 'VP',
  figures: [
    { key: 'pvs', title: 'Spot ', type: 'line' },
    { key: 'pvf', title: 'Fut ', type: 'line' },
  ],
  calc: (data) => data,
});

registerIndicator({
  name: 'VsBtc',
  figures: [
    { key: 'sb', title: 'Spot ', type: 'line' },
    { key: 'fb', title: 'Fut ', type: 'line' },
  ],
  calc: (data) => data,
});

const getJson = async (url) => {
  const f = await fetch(
    'https://dopa.thechadrs.com/api/proxy?url=' + btoa(url)
  );
  return f.json();
};

const getOI = async (ticker) => {
  const json = await getJson(
    'https://fapi.binance.com/futures/data/openInterestHist?period=30m&limit=500&symbol=' +
      ticker +
      'USDT'
  );
  let prev = +json[0].sumOpenInterestValue;
  return json.map((x, i) => {
    const v = Math.abs(+x.sumOpenInterestValue - prev);
    prev = +x.sumOpenInterestValue;
    return v;
  });
};

const getKlinesFut = async (ticker) => {
  const json = await getJson(
    'https://fapi.binance.com/fapi/v1/klines?interval=30m&limit=500&symbol=' +
      ticker +
      'USDT'
  );
  return json.map((x) => ({
    timestamp: x[0],
    open: +x[1],
    high: +x[2],
    low: +x[3],
    close: +x[4],
    volume: +x[5],
  }));
};

const getKlinesSpot = async (ticker) => {
  const json = await getJson(
    'https://api.binance.com/api/v3/klines?interval=30m&limit=500&symbol=' +
      ticker +
      'USDT'
  );
  return json.map((x) => ({
    timestamp: x[0],
    open: +x[1],
    high: +x[2],
    low: +x[3],
    close: +x[4],
    volume: +x[5],
  }));
};

const rocRatio = (xs, ys, period) => {
  const result = new Array(xs.length);
  let sx = 0,
    sy = 0;
  for (let i = 0; i < period; i++) {
    result[i] = 0;
    sx += xs[i];
    sy += ys[i];
  }

  for (let i = period; i < xs.length; i++) {
    const x = xs[i] / (sx / period);
    const y = ys[i] / (sy / period);

    result[i] = x - y;

    sx += xs[i] - xs[i - period];
    sy += ys[i] - ys[i - period];
  }
  return result;
};

const roc = (xs, period) => {
  const result = new Array(xs.length);
  let sx = 0;
  for (let i = 0; i < period; i++) {
    result[i] = 0;
    sx += xs[i];
  }

  for (let i = period; i < xs.length; i++) {
    const x = xs[i] / (sx / period) - 1;

    result[i] = x;

    sx += xs[i] - xs[i - period];
  }
  return result;
};

onMounted(async () => {
  const chart = init('chart');
  chart.setStyles({
    grid: { show: false },
  });

  chart.createIndicator('ROI', false, { height: 80 });
  chart.createIndicator('RC', false, { height: 80 });
  chart.createIndicator('VP', false, { height: 80 });
  chart.createIndicator('VsBtc', false, { height: 80 });

  const period = 24;
  const ticker = 'SOL';

  const fut = await getKlinesFut(ticker);
  const spot = await getKlinesSpot(ticker);
  const oi = await getOI(ticker);

  const fv = fut.map((x) => x.volume);
  const sv = spot.map((x) => x.volume);

  roc(oi, period).forEach((x, i) => {
    fut[i].ro = x;
  });
  roc(
    spot.map((x) => x.high - x.low),
    period
  ).forEach((x, i) => {
    fut[i].rp = x;
  });
  roc(sv, period).forEach((x, i) => {
    fut[i].rv = x;
  });
  rocRatio(
    sv,
    spot.map((x) => x.high - x.low),
    period
  ).forEach((x, i) => {
    fut[i].pvs = x;
  });
  rocRatio(
    fv,
    fut.map((x) => x.high - x.low),
    period
  ).forEach((x, i) => {
    fut[i].pvf = x;
  });
  const bf = await getKlinesFut('BTC');
  const bs = await getKlinesSpot('BTC');
  rocRatio(
    sv,
    bs.map((x) => x.volume),
    period
  ).forEach((x, i) => {
    fut[i].sb = x;
  });
  rocRatio(
    fv,
    bf.map((x) => x.volume),
    period
  ).forEach((x, i) => {
    fut[i].fb = x;
  });
  chart.applyNewData(fut);
});

onUnmounted(() => {
  // destroy chart
  dispose('chart');
});
</script>

<template>
  <div id="chart" style="width: 100%; height: 100vh" />
</template>
