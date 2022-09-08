# dmnt

このモジュールは、デバッグモニタを提供する Horizon OS のシステムモジュール `dmnt` を再実装したものです。

## 拡張機能

Atmosphère はチートコード機能を提供するための拡張機能を実装しています。

### チートサービス

HIPC サービス API は `dmnt:cht` サービスを通じたチートコードマネージャーとの対話を提供します。チートコードの記法に関する情報は [こちら](../../features/cheats.md) から。

`dmnt:cht` の SwIPC 定義はこのようになっています:

```
interface ams::dmnt::cheat::CheatService is dmnt:cht {
  [65000] HasCheatProcess() -> sf::Out<bool> out;
  [65001] GetCheatProcessEvent() -> sf::OutCopyHandle out_event;
  [65002] GetCheatProcessMetadata() -> sf::Out<CheatProcessMetadata> out_metadata;
  [65003] ForceOpenCheatProcess();
  [65004] PauseCheatProcess();
  [65005] ResumeCheatProcess();

  [65100] GetCheatProcessMappingCount() -> sf::Out<u64> out_count;
  [65101] GetCheatProcessMappings(u64 offset) -> sf::OutArray<MemoryInfo> &mappings, sf::Out<u64> out_count;
  [65102] ReadCheatProcessMemory(u64 address, u64 out_size) -> sf::OutBuffer &buffer;
  [65103] WriteCheatProcessMemory(sf::InBuffer &buffer, u64 address, u64 in_size);
  [65104] QueryCheatProcessMemory(u64 address) -> sf::Out<MemoryInfo> mapping;

  [65200] GetCheatCount() -> sf::Out<u64> out_count;
  [65201] GetCheats(u64 offset) -> sf::OutArray<CheatEntry> &cheats, sf::Out<u64> out_count;
  [65202] GetCheatById(u32 cheat_id) -> sf::Out<CheatEntry> cheat;
  [65203] ToggleCheat(u32 cheat_id);
  [65204] AddCheat(CheatDefinition &cheat, bool enabled) -> sf::Out<u32> out_cheat_id;
  [65205] RemoveCheat(u32 cheat_id);
  [65206] ReadStaticRegister(u8 which) -> sf::Out<u64> out;
  [65207] WriteStaticRegister(u8 which, u64 value);
  [65208] ResetStaticRegisters();

  [65300] GetFrozenAddressCount() -> sf::Out<u64> out_count;
  [65301] GetFrozenAddresses(u64 offset) ->sf::OutArray<FrozenAddressEntry> &addresses, sf::Out<u64> out_count;
  [65302] GetFrozenAddress(u64 address) -> sf::Out<FrozenAddressEntry> entry;
  [65303] EnableFrozenAddress(u64 address, u64 width) -> sf::Out<u64> out_value;
  [65304] DisableFrozenAddress(u64 address);
}
```
