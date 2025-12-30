## Recommended GameplayCameras + BP hookup plan (when you’re ready)

You already have the correct C++ contract for “no per-tick polling”:
- [ForceBoardCameraMode()](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Game/BrawlPlayerController.h:33:1-33:29) is called only when the desired camera “state” changes (active board / scouted board), and redundant applications are suppressed by `LastAppliedBoardCameraMode/Board/AnchorPlayerId`.

So the BP work should just *react* to that and let GameplayCameras handle blending.

### Existing BP assets detected
- [Content/Brawl/Blueprints/Game/BP_MatchPlayerController.uasset](cci:7://file:///E:/Unreal/BrawlFinal/Brawl/Content/Brawl/Blueprints/Game/BP_MatchPlayerController.uasset:0:0-0:0)
- [Content/Brawl/Blueprints/Game/BP_MatchPlayerState.uasset](cci:7://file:///E:/Unreal/BrawlFinal/Brawl/Content/Brawl/Blueprints/Game/BP_MatchPlayerState.uasset:0:0-0:0)

### Assets I recommend adding/using (names are suggestions; you can rename)
Since GameplayCameras isn’t in use yet, keep it clean and minimal:

#### 1) Camera Rig assets (GameplayCameras content)
- **`Content/Brawl/Blueprints/Camera/GC_Rig_Board.uasset`**
    - Top-down / board-framing rig.
    - Uses the board’s “view target” transform *as the anchor*.
- **`Content/Brawl/Blueprints/Camera/GC_Rig_OTS.uasset`** (if you don’t already have an OTS camera mode/rig)
    - Your normal “player avatar” rig.

If your GameplayCameras setup uses a “Camera Asset” + “Camera Rigs” + “Director”, adapt names accordingly—the key is: one rig for board view.

#### 2) A camera “driver” component or logic location
Pick one location (keep it simple):
- **Option A (recommended):** put the GameplayCameras orchestration on the **player pawn / avatar BP** (since it’s the thing that owns the camera).
- `BP_MatchPlayerController` just calls into the pawn when [ForceBoardCameraMode](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Game/BrawlPlayerController.h:33:1-33:29) fires.

This matches your desire: “swap from player character’s OTS GameplayCamera to board camera”.

---

## Exact wiring (Blueprint)

### In `BP_MatchPlayerController` implement [ForceBoardCameraMode](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Game/BrawlPlayerController.h:33:1-33:29)
**Goal:** switch to board camera mode on the local client, but do not decide any gameplay.

Suggested BP behavior:
1) `Get Pawn`
2) If valid:
    - Call an interface or BP event on the pawn, e.g. `BP_ApplyBoardCameraMode` (you can add this event to your pawn BP)
3) If not valid (spectator):
    - Do nothing; C++ already handles `SetViewTargetWithBlend` using the board transform and your new tuning vars.

### In the Player Avatar BP (your pawn)
Implement `BP_ApplyBoardCameraMode`:
- Activate GameplayCameras “Board rig” (and rely on your internal blend rules inside the rig assets).
- Ensure mouse cursor/UI mode remains correct (if you manage that here).

**Anchor transform source:**
- You *don’t* need to recompute anything in BP if you don’t want to—C++ is already positioning spectator cameras using:
    - [Board->TryGetViewTargetTransformForPlayer(PlayerId)](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlGrid/Public/Actors/BrawlBoardActor.h:126:1-126:94)

For the pawn’s GameplayCameras rig, you’ll need an equivalent anchor. Minimal options:
- If the pawn knows which board it’s on (teleported to presence anchor), you can just drive the rig from pawn location.
- Or (better, matches your prompt): query the board for a transform anchor (but that requires the pawn to know which board/player-id it is framing).

Since you asked for “anchored on [Board->TryGetViewTargetTransformForPlayer(TargetPlayerId)](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlGrid/Public/Actors/BrawlBoardActor.h:126:1-126:94)”, I’d keep the anchor computed in C++ and exposed to BP only if needed. **Right now C++ does not pass the transform into BP**, it only calls [ForceBoardCameraMode()](cci:1://file:///E:/Unreal/BrawlFinal/Brawl/Source/BrawlMatch/Public/Game/BrawlPlayerController.h:33:1-33:29).

So there’s one optional follow-up if you want the pawn’s camera rig to be board-anchored deterministically without doing BP searches:
- Add a **client-only BP-callable getter** on the controller like `GetDesiredBoardCameraViewTransform(out Transform, out bIsScouting, out AnchorPlayerId)`.

I did **not** add that yet because you said “keep C++ minimal” and you’re ~2 weeks from hooking up GameplayCameras; we can add it exactly when you know what the rig expects.

---

## WBP binding directions (what you need to do)
In your scouting/scoreboard widget(s) that render player rows:

### Bind disabled/enabled state
- Bind the row button’s `IsEnabled` (or equivalent) to:
    - `Entry.bCanBeScouted`

### Bind click -> request
On click for a given `UBrawlVM_ScoutingPlayerEntry` row:
- Call [UBrawlVM_Scouting::RequestScoutPlayer(Entry.PlayerId)](cci:1://file:///e:/Unreal/BrawlFinal/Brawl/Source/BrawlUI/Public/UI/BrawlVM_Scouting.h:141:1-141:47)

Behavior you’ll see:
- If the clicked `PlayerId == LocalPlayerId`, server clears scouting and you’ll return to **Active** board (combat arena) via existing fallback logic.
- If target is spectator/eliminated, UI should already be disabled; server would also reject.

### Board list selection (optional)
If you have a separate board list:
- Keep using [UBrawlVM_Scouting::SelectBoardByIndex()](cci:1://file:///e:/Unreal/BrawlFinal/Brawl/Source/BrawlUI/Public/UI/BrawlVM_Scouting.h:135:1-135:43) / [SetSelectedBoardActor()](cci:1://file:///e:/Unreal/BrawlFinal/Brawl/Source/BrawlUI/Public/UI/BrawlVM_Scouting.h:138:1-138:60).
- Note: selection will now be more stable during transfer due to cached last scouted board.

---
