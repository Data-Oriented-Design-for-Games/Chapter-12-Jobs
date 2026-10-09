# Chapter 12 — Jobs

Sample project for **Chapter 12** of [*High Performance Unity Game Development (Using data-oriented design)*](https://www.manning.com/books/high-performance-unity-game-development) by Nitzan Wilnai (Manning).

This is the survivor game from [Chapter 11](https://github.com/Data-Oriented-Design-for-Games/Chapter-11). The player stands in the middle of the screen. Zombies spawn on a circle around the player and walk toward it. This variant moves the enemies with the Unity C# Job System, so the movement loop is split across worker threads instead of running as one loop on the main thread.

## What it shows

- How to turn a plain `for` loop over game data into an `IJobParallelFor`.
- Why the data moves from managed arrays to `NativeArray` first.
- `[ReadOnly]` and `[NativeDisableParallelForRestriction]` on job fields.
- Scheduling a job and waiting for it in the same frame with `Complete()`.
- A built-in timing test, so you can compare this sample with its three siblings.

## What changed from [Chapter 11](https://github.com/Data-Oriented-Design-for-Games/Chapter-11)

- `GameData.cs`: the enemy arrays are now native. `EnemyPosition` is a `NativeArray<float2>` (it was a `Vector2[]`). `EnemyType`, `AliveEnemyIndices` and `DeadEnemyIndices` are `NativeArray<int>`. `EnemyVelocityNative` is a native copy of `balance.EnemyVelocity`, so the job can read enemy speeds.
- `Logic.cs`: `AllocateGameData` allocates the arrays with `Allocator.Persistent`. The new `FreeGameData` disposes them, and `Game.OnDestroy` calls it.
- `Logic.cs`: the `moveEnemies` loop is gone. `MoveEnemiesJob` replaces it:

  ```csharp
  [BurstCompile]
  struct MoveEnemiesJob : IJobParallelFor
  {
      // fields: AliveEnemyIndices, EnemyType, EnemyVelocity, EnemyPosition, Dt

      public void Execute(int i)
      {
          int    enemyIndex = AliveEnemyIndices[i];
          float2 pos        = EnemyPosition[enemyIndex];
          float2 dir        = -math.normalizesafe(pos);
          float  speed      = EnemyVelocity[EnemyType[enemyIndex]];
          EnemyPosition[enemyIndex] = pos + dir * speed * Dt;
      }
  }
  ```

- `Logic.Tick` schedules the job with `moveJob.Schedule(gameData.AliveEnemyCount, 64)` and calls `moveHandle.Complete()` right after.
- The job is marked `[BurstCompile]`, so this sample uses Burst as well. Chapter-12-Burst shows Burst without jobs.
- The rest of the math in `Logic.cs` uses `Unity.Mathematics` (`float2`, `math.lengthsq`, `math.normalizesafe`) instead of `Vector2`.
- Only enemy movement is a job. Spawning, `checkEnemyOutOfBounds`, `doEemyToEnemyCollision` and `movePlayer` are still main-thread loops. `Board.Tick` still copies each position to its `Transform` on the main thread.
- `Board.cs` adds a `TransformAccessArray` (`m_enemyTransforms`) and an `m_poolToEnemyIndex` array and keeps them up to date. No job reads them in this sample. Chapter-12-Jobs-Transforms uses them.
- `Game.cs` adds `runPerformanceTest`, which runs when you press **T** on the main menu.
- In `Logic.Tick` the call to `checkGameOver` is commented out, so an enemy touching the player does not end the game.

## The four Chapter 12 samples

All four start from the same game and change the same step: moving the enemies.

| Sample | What it does |
| --- | --- |
| [Chapter-12-Jobs](https://github.com/Data-Oriented-Design-for-Games/Chapter-12-Jobs) | Moves the enemies in a Burst-compiled `IJobParallelFor` on worker threads. Transforms are still written on the main thread. |
| [Chapter-12-Jobs-Transforms](https://github.com/Data-Oriented-Design-for-Games/Chapter-12-Jobs-Transforms) | Chapter-12-Jobs plus a second job, an `IJobParallelForTransform`, that also writes the enemy transforms on worker threads. |
| [Chapter-12-Burst](https://github.com/Data-Oriented-Design-for-Games/Chapter-12-Burst) | No jobs. The movement loop is a Burst-compiled static method called on the main thread. |
| [Chapter-12-ECS](https://github.com/Data-Oriented-Design-for-Games/Chapter-12-ECS) | Stores each enemy as an entity (Unity Entities package) and moves it through `EntityManager` on the main thread. |

Jobs, Burst and ECS are three separate alternatives, not steps in a sequence. Each one starts from the same base: the [Chapter 11](https://github.com/Data-Oriented-Design-for-Games/Chapter-11) game with its enemy arrays changed to `NativeArray`. Jobs + Transforms is the only sample that builds on a sibling (Jobs).

## How the code is organized

Everything is in `Assets/Scripts`.

- `Game.cs` — entry point. Owns `GameData`, `MetaData` and `Balance`, switches between menu states, calls `Board.Tick` every frame.
- `GameData.cs` — the game state: enemy arrays, alive and dead index lists, timers.
- `Logic.cs` — static functions that change `GameData`, plus `MoveEnemiesJob`.
- `Board.cs` — the link to Unity: input, the pool of enemy GameObjects, copying positions to transforms.
- `Balance.cs` — tuning data, loaded from `Assets/Resources/balance.bytes`.
- `BalanceParser.cs` — editor tool that builds `balance.bytes` from the ScriptableObjects in `Assets/Data`.
- `GameDataIO.cs`, `MetaDataIO.cs` — binary save and load.
- `MainMenuVisual.cs`, `PauseMenuVisual.cs`, `GameOverVisual.cs` — the UI screens.

## Running it

1. Open the project in Unity **6000.3.22f1** (Unity 6.3 LTS) or newer. On first open Unity downloads the packages in `Packages/manifest.json`, including Burst, Collections and Mathematics.
2. Open `Assets/Scenes/MainGameScene.unity` and press **Play**.
3. Click **New game**. Hold the left mouse button and drag to steer. The drag direction, measured from the point where you pressed, is the direction the player moves. Outside the Editor the code reads touch input instead.
4. On the main menu, press **T** to run the timing test. It starts a game, calls `Board.Tick(0.016f)` 1,000 times in a row, and shows the total and per-call time on screen and in the Console. All four samples have the same test, so you can compare them on your own machine.
5. Burst has to be on: **Jobs > Burst > Enable Compilation** (on by default). In the Editor, Burst compiles in the background the first time the code runs, so run the test twice and use the second result.
6. Press **S** to save a screenshot (`screenshot0.png`, `screenshot1.png`, ...).

If you change `Assets/Data/Balance.asset` or the enemy assets in `Assets/Data/Enemies`, run **DOD > Balance > Parse Local** to rebuild `Assets/Resources/balance.bytes`. The game reads that file, not the ScriptableObjects.

## More samples

All sample projects for the book: https://github.com/Data-Oriented-Design-for-Games
