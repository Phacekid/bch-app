<script setup>
import HelloWorld from './components/HelloWorld.vue';
import Nftitem from './components/Nftitem.vue';
import { ref, onMounted } from 'vue';
import { ElectrumClient, ElectrumTransport } from 'electrum-cash';
import { vmNumberToBigInt, hexToBin } from '@bitauth/libauth';

const ninjas = ref();
const blockHeight = ref();

onMounted(async () => {
  const electrum = new ElectrumClient(
    'Electrum client example',
    '1.4.1',
    'bch.imaginary.cash',
    ElectrumTransport.WSS.Port,
    ElectrumTransport.WSS.Scheme
  );

  await electrum.connect();

  function handleNewBlocks(data) {
    const currentBlockHeight = data.height;
    // console.log(data);
    // console.log(currentBlockHeight);
    blockHeight.value = currentBlockHeight;
  }

  await electrum.subscribe(handleNewBlocks, 'blockchain.headers.subscribe');
  const mintAddress = "bitcoincash:pvl88yaeyajf6cn8fhaf3qasl238rzrnpmnmgzr05krpkt0ez52z5rgwl9x9u";
  const response = await electrum.request('blockchain.address.get_history', mintAddress);
  // console.log(response);
  const recentMints = response.slice(-10);
  // console.log(recentMints);
  const listPromises = [];
  console.time('Execution Time');

  for (const tx of recentMints) {
    const txInfoPromise = electrum.request(
      'blockchain.transaction.get',
      tx['tx_hash'],
      true
    );
    listPromises.push(txInfoPromise);
  }
  const txInfoList = await Promise.all(listPromises);
  console.timeEnd('Execution Time');
  const ninjaList = [];

  for (const txInfo of txInfoList) {
    const nftCommitment = txInfo["vout"][1]["tokenData"]["nft"]["commitment"];
    ninjaList.push({
      commitment: nftCommitment,
      nftNumber: vmNumberToBigInt(hexToBin(nftCommitment)) + 1n
    });
  }

  ninjas.value = ninjaList.reverse();

});
</script>

<template>
  <header>
    <img alt="Vue logo" class="logo" src="./assets/logo.svg" width="125" height="125" />

    <div class="wrapper">
      <HelloWorld msg="First BCH APP!" />
      <h3>Current BlockHeight: {{ blockHeight }}</h3>
    </div>
  </header>

  <main style="margin-top: 50px; display: flex; flex-wrap: wrap;">
    <div v-for="ninja in ninjas">
      <Nftitem :commitment="ninja.commitment" :nftNumber="ninja.nftNumber" />
    </div>
  </main>
</template>

<style scoped>
header {
  line-height: 1.5;
}

.logo {
  display: block;
  margin: 0 auto 2rem;
}

@media (min-width: 1024px) {
  header {
    display: flex;
    place-items: center;
    padding-right: calc(var(--section-gap) / 2);
    width: 1000px;
  }

  .logo {
    margin: 0 2rem 0 0;
  }
}
</style>
