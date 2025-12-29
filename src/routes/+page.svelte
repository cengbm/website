<script lang="ts">
  import {onMount} from 'svelte';
  import { PUBLIC_UP_ADDR } from '$env/static/public';

  import { ethers } from "ethers";
  import { ERC725 } from "@erc725/erc725.js";
  import LSP6Schema from "@erc725/erc725.js/schemas/LSP6KeyManager.json";
  import UniversalProfileArtifact from "@lukso/lsp-smart-contracts/artifacts/UniversalProfile.json";


  const myUniversalProfileAddress = "0xc1D729fE470E065Ec09393b656756185729e43E0";
  const RPC_ENDPOINT = 'https://rpc.testnet.lukso.network';
  const TRACKER_ADDRESS = "0xc2A5D274e88235460b4FcB5748AD1eAD23f3e3Cb";

  let erc725 : ERC725;
   
  onMount(() => {
    erc725 = new ERC725(LSP6Schema, myUniversalProfileAddress, RPC_ENDPOINT);
  })

  
 
 

  async function grantPermissions() {


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

  const provider = new ethers.BrowserProvider(window.lukso);
  const signer = await provider.getSigner();
  const accounts = await provider.send('eth_requestAccounts', []);

  const myUP = new ethers.Contract(
    accounts[0],
    UniversalProfileArtifact.abi,
    signer
  );

  try{
    await myUP
    .setData(permissionData.keys,permissionData.values);

    const updatedPermissions = await erc725.getData({
      keyName: 'AddressPermissions:Permissions:<address>',
      dynamicKeyParts: TRACKER_ADDRESS,
    });

    if (updatedPermissions && typeof updatedPermissions.value == 'string') {
      console.log(
        'This shit worked son:',
          erc725.decodePermissions('0x ${updatedPermissions.value}'),
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
