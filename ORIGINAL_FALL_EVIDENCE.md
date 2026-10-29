# Original Gothic II lethal-fall evidence

Base executable inspected: original Gothic II Addon `Gothic2.exe` from the user's second Google Drive.

## ZenGin API mapping

- `oCAniCtrl_Human::StartFallDownAni` = `0x006B5220`
- `oCAniCtrl_Human::CheckFallStates` = `0x006B5810`
- `oCNpc::CreateFallDamage(float)` = `0x00681DA0`
- `oCNpc::OnDamage_Anim(oSDamageDescriptor&)` = `0x00675BD0`
- `oEDamageType_Fall` = `0x80`
- `oEDamageDescFlag_DamageType` = `0x80`
- `oEDamageDescFlag_Damage` = `0x200`

## Landing order

`CheckFallStates()` starts `T_FALL_2_FALLEN` / `T_FALLB_2_FALLENB` first and only afterwards calls `CreateFallDamage(float)`.

Therefore the fall controller owns the landing/fallen animation before the damage pipeline runs.

## Fall descriptor

`CreateFallDamage(float)` builds a damage descriptor with:

- `dwFieldsValid = 0x280` = DamageType + Damage
- `enuModeDamage = 0x80` = `oEDamageType_Fall`

## Animation behavior

Inside `OnDamage_Anim()`, the function reads `descriptor.enuModeDamage` at offset `+0x24`.
At `0x0067680E` it masks the value with `0x80` and keeps the result as the FALL condition.
At `0x00676B69` the FALL condition is tested; when it is set, execution bypasses the normal hit/death animation-selection branch and proceeds to function cleanup/return.

This is the important semantic difference from stock OpenGothic 0.6.7: OpenGothic computes fall damage separately, selects `FallenA/FallenB`, then loses the damage cause when HP reaches zero and `onNoHealth()` unconditionally selects `DeadA/DeadB`.

## Isolated FIX25 patch

Repository: `Erken22/OpenGothicFallDeath`

The patch carries one transient `isFallDamage` flag through:
`takeFallDamage -> changeAttribute -> checkHealth -> onNoHealth`.

Only fall damage passes `true`. When lethal FALL reaches `onNoHealth`, all normal death-state work still runs, but `DeadA/DeadB` is not started. Other damage/death sources retain the existing OpenGothic behavior.

This mirrors ZenGin's explicit FALL damage animation suppression rather than inferring behavior from the current pose.
