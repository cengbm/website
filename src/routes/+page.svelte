<script lang="ts">
  import {onMount} from 'svelte';
  import { PUBLIC_UP_ADDR} from '$env/static/public';

  import { ethers } from "ethers";
  import { ERC725 } from "@erc725/erc725.js";
  import LSP6Schema from "@erc725/erc725.js/schemas/LSP6KeyManager.json";
  import LSP6KeyManagerArtifact from "@lukso/lsp-smart-contracts/artifacts/LSP6KeyManager.json";
  import UniversalProfileArtifact from "@lukso/lsp-smart-contracts/artifacts/UniversalProfile.json";


  const myUniversalProfileAddress = "0x5aF1D0DF0ae69E9c2A49748A158FAb468A62c09e";
  const RPC_ENDPOINT = 'https://rpc.testnet.lukso.network';
  const TRACKER_ADDRESS = "0xc2A5D274e88235460b4FcB5748AD1eAD23f3e3Cb";

  let erc725 : ERC725;
   


  //++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++//

  async function readControllerPermissions(controller: string) {
  const erc725 = new ERC725(
    LSP6Schema,
    myUniversalProfileAddress,          // dein UP
    RPC_ENDPOINT
  );

  const res = await erc725.getData({
    keyName: 'AddressPermissions:Permissions:<address>',
    dynamicKeyParts: controller,
  });

  console.log('Controller:', controller);
  console.log('Raw value:', res.value);

  if (res.value) {
    console.log(
      'Decoded:',
      erc725.decodePermissions(res.value as `0x${string}`)
    );
  } else {
    console.log('❌ No permissions set');
  }
}

async function readPerm() {
  await readControllerPermissions('0x7870C5B8BC9572A8001C3f96f7ff59961B23500D');
await readControllerPermissions('0x69843800406517DADEf2B5F06A7D5eF81b44CE34');

}

  async function readPermissions() {
     const provider = new ethers.BrowserProvider(window.lukso);
  const accounts = await provider.send('eth_requestAccounts', []);
  const signer = await provider.getSigner();

    const erc725 = new ERC725(
    LSP6Schema,
    accounts[0],
    RPC_ENDPOINT
  );
const controllers = await erc725.getData('AddressPermissions[]');
console.log('Controllers:', controllers.value);
  }

  //++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++//
 /* onMount(() => {
    erc725 = new ERC725(LSP6Schema, myUniversalProfileAddress, RPC_ENDPOINT);
  })
*/
  
 /* async function grantPermissionsWithKM() {
   

 const TRACKER_ADDRESS = "0xc2A5D274e88235460b4FcB5748AD1eAD23f3e3Cb";


 const provider = new ethers.BrowserProvider(window.lukso);
  const accounts = await provider.send('eth_requestAccounts', []);
  const signer = await provider.getSigner();


const up = new ethers.Contract(
  accounts[0],
  UniversalProfileArtifact.abi,
  provider
);

const keyManagerAddress = await up.owner();

const keyManager = new ethers.Contract(
  keyManagerAddress,
  LSP6KeyManagerArtifact.abi,
  signer,
);

   const permissionSetAnyDataKey = erc725.encodePermissions({
    SETDATA: true,
    });
  
    const addressPermissionsArrayValue = await erc725.getData(
    'AddressPermissions[]',
  );

  let numberOfController = 0;

  if(Array.isArray(addressPermissionsArrayValue.value)) {
    numberOfController = addressPermissionsArrayValue.value.length;
  }

  const permissionData = erc725.encodeData([
      {
    keyName: 'AddressPermissions:Permissions:<address>',
    dynamicKeyParts: TRACKER_ADDRESS,
    value: permissionSetAnyDataKey,
  },
  // The `AddressPermissions[]` array contains the list of permissioned addresses (= controllers)
  // This adds the `newControllerAddress` in this list at the end (at the last index) and increment the array length.
  {
    keyName: 'AddressPermissions[]',
    value: [TRACKER_ADDRESS],
    startingIndex: numberOfController,
    totalArrayLength: numberOfController + 1,
  },
  ]);

  const upInterface = new ethers.Interface(
  UniversalProfileArtifact.abi
);

const payload = upInterface.encodeFunctionData('setData', [
  permissionData.keys,
  permissionData.values,
]);

const tx = await keyManager.execute(payload);
await tx.wait();

console.log('✅ TRACKER_ADDRESS hat jetzt SETDATA Permission');


const permissionKey = erc725.encodeKeyName(
  'AddressPermissions:Permissions:<address>',
  TRACKER_ADDRESS
);

const storedPermissions = await up.getData(permissionKey);
console.log(storedPermissions);


  }
 
 */

import {
  keccak256,
  toUtf8Bytes,
  zeroPadValue,
  concat,
  hexlify,
} from 'ethers';

export async function grantSetDataPermission() {
  // 1️⃣ Universal Profile holen
  const [upAddress] = await window.lukso.request({
    method: 'eth_requestAccounts',
  });
const erc725 = new ERC725(
    LSP6Schema,
    upAddress,
    RPC_ENDPOINT
  );

  // 3️⃣ Permission Bits
  const permissionSetAnyDataKey = erc725.encodePermissions({
    SETDATA: true,
  });

  // 4️⃣ Aktuelle Controller-Anzahl
  const addressPermissionsArrayValue = await erc725.getData(
    'AddressPermissions[]',
  );

  const numberOfControllers = Array.isArray(addressPermissionsArrayValue.value)
    ? addressPermissionsArrayValue.value.length
    : 0;

  // 5️⃣ OFFIZIELLES Encoding (Keys + Values)
  const permissionData = erc725.encodeData([
    {
      keyName: 'AddressPermissions:Permissions:<address>',
      dynamicKeyParts: TRACKER_ADDRESS,
      value: permissionSetAnyDataKey,
    },
    {
      keyName: 'AddressPermissions[]',
      value: [TRACKER_ADDRESS],
      startingIndex: numberOfControllers,
      totalArrayLength: numberOfControllers + 1,
    },
  ]);

  // 6️⃣ ABI-Encoding von setData(bytes32[],bytes[])
  const iface = new ethers.Interface(
    UniversalProfileArtifact.abi
  );

  const calldata = iface.encodeFunctionData(
    'setData(bytes32,bytes)',
    [permissionData.keys, permissionData.values]
  );

  // 7️⃣ Senden über EIP-1193 (Extension rewritet → Key Manager)
  const txHash = await window.lukso.request({
    method: 'eth_sendTransaction',
    params: [
      {
        from: upAddress,
        to: upAddress,
        data: calldata,
        value: '0x0',
      },
    ],
  });

  console.log('✅ SETDATA Permission vergeben (offiziell)');
  console.log('TX Hash:', txHash);

  return txHash;
}

async function setPermissions(){
  const scheme = [{
    'name' : 'AddressPermissions:Permissions:<address>',
    'key': "0x4b80742de2bf82acb3630000<address>",
    'keyType': 'MappingWithGrouping',
    'valueType': 'bytes32',
    'valueContent': 'BitArray',
  }];

  const provider = new ethers.BrowserProvider(window.lukso);
  const accounts = await provider.send('eth_requestAccounts', []);
  const signer = await provider.getSigner();

  const erc725 = new ERC725(
    scheme,
    accounts[0],
    RPC_ENDPOINT,
  );

  const permissionSetAnyDataKey = erc725.encodePermissions({
    SETDATA: true,
    });

  const permissionData = erc725.encodeData([
      {
    keyName: 'AddressPermissions:Permissions:<address>',
    dynamicKeyParts: TRACKER_ADDRESS,
    value: permissionSetAnyDataKey,
  },
  ], scheme);  

  const myUP = new ethers.Contract(
    accounts[0],
    UniversalProfileArtifact.abi,
    signer,
  );

  await myUP.setDataBatch(permissionData.keys, permissionData.values);
}


async function testSets(){
  const scheme = [{
    'name': 'AddressPermissions[]',
    'key': '0xdf30dba06db6a30e65354d9a64c609861f089545ca58c6b4dbe31a5f338cb0e3',
    'keyType': 'Array',
    'valueType': 'address',
    'valueContent': 'Address',
  }];

  const provider = new ethers.BrowserProvider(window.lukso);
  const accounts = await provider.send('eth_requestAccounts', []);
  const signer = await provider.getSigner();

  console.log(signer);

  console.log('PLATZ MINIMUM');

  const erc725 = new ERC725(
    scheme,
    accounts[0],
    RPC_ENDPOINT,
  );

   const addressPermissionsArrayValue = await erc725.getData(
    'AddressPermissions[]',
  );

  let numberOfController = 0;

  if(Array.isArray(addressPermissionsArrayValue.value)) {
    numberOfController = addressPermissionsArrayValue.value.length;
  }

  console.log(numberOfController);

  const check = erc725.encodeData(
    [
      {
        keyName: 'AddressPermissions[]',
        value: [TRACKER_ADDRESS],
        startingIndex: numberOfController,
        totalArrayLength: numberOfController + 1,
      }
    ],
    scheme
  );


  console.log("check");
  console.log(check);

  const myUP = new ethers.Contract(
    accounts[0],
    UniversalProfileArtifact.abi,
    signer,
  );

      console.log("MYUP");
      console.log(myUP.runner);

  await myUP.setDataBatch(check.keys,check.values);
  
}


  async function grantPermissions() {


  const provider = new ethers.BrowserProvider(window.lukso);
  const accounts = await provider.send('eth_requestAccounts', []);
  const signer = await provider.getSigner();

    const erc725 = new ERC725(
    LSP6Schema,
    accounts[0],
    RPC_ENDPOINT
  );


    const updatedPermissions = await erc725.getData({
      keyName: 'AddressPermissions:Permissions:<address>',
      dynamicKeyParts: TRACKER_ADDRESS,
    });
  
    const permissionSetAnyDataKey = erc725.encodePermissions({
    SETDATA: true,
    });
  
    const addressPermissionsArrayValue = await erc725.getData(
    'AddressPermissions[]',
  );

  let numberOfController = 0;

  if(Array.isArray(addressPermissionsArrayValue.value)) {
    numberOfController = addressPermissionsArrayValue.value.length;
  }

  const permissionData = erc725.encodeData([
      {
    keyName: 'AddressPermissions:Permissions:<address>',
    dynamicKeyParts: TRACKER_ADDRESS,
    value: permissionSetAnyDataKey,
  },
  ]);


  const signerAddr = await signer.getAddress();

  console.log(signer);
  console.log(signerAddr);
  

  const myUP = new ethers.Contract(
    accounts[0],
    UniversalProfileArtifact.abi,
    provider
  );

  console.log(accounts[0]);

  try{
    await myUP.connect(accounts[0])
    //@ts-expect-error 
    .setData(permissionData.keys,permissionData.values);

    const updatedPermissions = await erc725.getData({
      keyName: 'AddressPermissions:Permissions:<address>',
      dynamicKeyParts: TRACKER_ADDRESS,
    });


    if (updatedPermissions && typeof updatedPermissions.value == 'string') {
      console.log(
        'This shit worked son:',
          erc725.decodePermissions(updatedPermissions.value as `0x${string}`),
      );
    } else {
      console.error(
        'No Permissions for this address '
      )
    }
  } catch (error) {
    console.error("DA GEHT NISCH", error);
  }

  }
  
</script>


<h1>KYSA</h1>

<button
  aria-label="Grant tracker permissions"
  onclick={grantPermissions}
>
  Grant Tracker Permissions
</button>

<button
  aria-label="testPresence"
  onclick={setPermissions}
>
  Test Presence
</button>


<button
  aria-label="readPresence"
  onclick={testSets}
>
  Test READ
</button>

