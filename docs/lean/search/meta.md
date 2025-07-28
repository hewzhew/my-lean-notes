# 元编程搜索

通过元编程也可以从已在环境中注册过的常量中快速搜索我们想要的定理。

以下代码由大佬@sinianluoye 提供，在此表示感谢

```lean
import Lean

open Lean
open Lean.Elab
open Lean.Elab.Command

/--
`find_thms "xxx" "yyy" "-zzz" …`
在当前环境中查找包含 xxx 且包含 yyy 但不包含 zzz 的定理。
定理的分词是按 `.` 和 `_` 分割的
-/
syntax (name := findThms) "#find_thms" (str)+ : command

@[command_elab findThms]
def elabFindThms : CommandElab := fun stx => do
  let args := stx[1].getArgs.map fun s =>
    s.isStrLit?
    |>.get!

  let env ← getEnv

  let filter_name := fun (nm: Name) =>
    let nameStr := nm.toString
    let p1 := nameStr.splitOn "."
    let parts := p1.map (fun s => s.splitOn "_")
    ¬ nm.isInternal ∧ args.all fun a =>
      if a.startsWith "-"
      then parts.flatten.all fun p => (a.toSubstring.drop 1).toString ≠ p
      else parts.flatten.any fun p => a = p


  let thms :=
    env.constants.toList.foldl (init := #[]) fun acc (nm, cinfo) =>
      match cinfo with
      | ConstantInfo.thmInfo .. => acc.push (nm, cinfo)
      | _ => acc

  let res := thms.filter (fun (nm, _) => filter_name nm)

  if res.isEmpty then
    logInfo m!"[find_thms] no matching theorems"
  else
    for item in res do
      let info := item.2
      let msg := match info with
      | ConstantInfo.thmInfo val => val.type
      | _ => panic! "unexpected constant info type"
      logInfo m!"{item.1}:\n{msg}"

#find_thms "gaussian" "col" "-aux"

```

此外，我又根据学习lean的需求改写了一份用于查看已有定理依赖的代码

```lean
import Lean
import Std.Data.HashSet
import Lean.Data.Json

open Lean
open Lean.Elab
open Lean.Elab.Command
open Std (HashSet HashMap)

-- 依赖层级结构
structure DependencyLayer where
  depth : Nat
  theorems : Array Name
deriving Repr, ToJson, FromJson

-- 依赖关系数据结构
structure DependencyGraph where
  rootTheorem : Name
  layers : Array DependencyLayer
  edges : HashMap Name (Array Name)  -- 每个定理的直接依赖
deriving Repr, Nonempty

-- 提取表达式中引用的所有常量名
def references (expr : Expr) : HashSet Name :=
  let rec go (e : Expr) (acc : HashSet Name) : HashSet Name :=
    match e with
    | .const name _ => acc.insert name
    | .app f arg => go arg (go f acc)
    | .lam _ type body _ => go body (go type acc)
    | .forallE _ type body _ => go body (go type acc)
    | .letE _ type value body _ => go body (go value (go type acc))
    | .mdata _ expr => go expr acc
    | .proj _ _ struct => go struct acc
    | _ => acc
  go expr ∅

-- 检查常量是否为定理的辅助函数
def isTheorem (env : Environment) (name : Name) : Bool :=
  match getOriginalConstKind? env name with
  | some (.thm) => true
  | _ => false

-- 获取定理的直接依赖
def getDirectDependencies (env : Environment) (theoremName : Name) : Array Name :=
  match env.find? theoremName with
  | some constInfo =>
    let typeRefs := references constInfo.type
    let valueRefs := constInfo.value?.map references |>.getD ∅
    let allRefs := typeRefs ∪ valueRefs
    -- 过滤掉内置类型、当前定理本身，只保留定理
    allRefs.toArray.filter fun name =>
      name != theoremName &&
      !name.isInternal &&
      isTheorem env name
  | none => #[]

/--
`#find_deps "theorem_name"`
查找指定定理的所有依赖定理
-/
syntax (name := findDeps) "#find_deps" str : command

@[command_elab findDeps]
def elabFindDeps : CommandElab := fun stx => do
  let theoremNameStr := stx[1].isStrLit?.get!
  let theoremName := theoremNameStr.toName

  let env ← getEnv

  -- 检查定理是否存在
  match env.find? theoremName with
  | none =>
    logError m!"Theorem '{theoremName}' not found"
    return
  | some constInfo =>
    if !isTheorem env theoremName then
      logError m!"'{theoremName}' is not a theorem"
      return

    let deps := getDirectDependencies env theoremName

    if deps.isEmpty then
      logInfo m!"[find_deps] '{theoremName}' has no theorem dependencies"
    else
      logInfo m!"[find_deps] Dependencies of '{theoremName}':"
      for dep in deps.qsort Name.lt do
        match env.find? dep with
        | some depInfo =>
          let depType := match depInfo with
          | ConstantInfo.thmInfo val => val.type
          | _ => panic! "unexpected constant info type"
          logInfo m!"  {dep}: {depType}"
        | none => continue

/--
`#find_all_deps "theorem_name"`
递归查找指定定理的所有依赖定理（包括间接依赖）
-/
syntax (name := findAllDeps) "#find_all_deps" str : command

/--
`#find_deps_depth "theorem_name" depth`
查找指定定理的依赖定理，限制搜索深度，按层级显示
-/
syntax (name := findDepsDepth) "#find_deps_depth" str num : command

/--
`#export_deps_json "theorem_name" depth "filename"`
导出依赖关系到JSON文件供Python可视化
-/
syntax (name := exportDepsJson) "#export_deps_json" str num str : command

-- 递归获取所有依赖
partial def getAllDependencies (env : Environment) (theoremName : Name) (visited : HashSet Name := ∅) : HashSet Name :=
  if visited.contains theoremName then
    visited
  else
    let directDeps := getDirectDependencies env theoremName
    let newVisited := visited.insert theoremName
    directDeps.foldl (fun acc dep => getAllDependencies env dep acc) newVisited

-- 获取指定深度的依赖（按层级组织）
partial def getDependenciesByLayers (env : Environment) (theoremName : Name) (maxDepth : Nat) : DependencyGraph :=
  let rec buildLayers (currentLayer : Array Name) (depth : Nat) (visited : HashSet Name) (edges : HashMap Name (Array Name)) (layers : Array DependencyLayer) : DependencyGraph :=
    if depth >= maxDepth || currentLayer.isEmpty then
      { rootTheorem := theoremName, layers := layers, edges := edges }
    else
      let (nextLayer, newVisited, newEdges) := currentLayer.foldl (fun (nextAcc, visitedAcc, edgesAcc) thm =>
        if visitedAcc.contains thm then
          (nextAcc, visitedAcc, edgesAcc)
        else
          let newVisitedAcc := visitedAcc.insert thm
          let deps := getDirectDependencies env thm
          let newEdgesAcc := edgesAcc.insert thm deps
          let newNextAcc := deps.foldl (fun acc dep =>
            if !newVisitedAcc.contains dep then acc.push dep else acc) nextAcc
          (newNextAcc, newVisitedAcc, newEdgesAcc)
      ) (#[], visited, edges)

      let currentLayerData : DependencyLayer := { depth := depth, theorems := currentLayer }
      let newLayers := layers.push currentLayerData
      buildLayers nextLayer (depth + 1) newVisited newEdges newLayers

  buildLayers #[theoremName] 0 ∅ ∅ #[]

@[command_elab findAllDeps]
def elabFindAllDeps : CommandElab := fun stx => do
  let theoremNameStr := stx[1].isStrLit?.get!
  let theoremName := theoremNameStr.toName

  let env ← getEnv

  -- 检查定理是否存在
  match env.find? theoremName with
  | none =>
    logError m!"Theorem '{theoremName}' not found"
    return
  | some constInfo =>
    if !isTheorem env theoremName then
      logError m!"'{theoremName}' is not a theorem"
      return

    let allDeps := getAllDependencies env theoremName
    let filteredDeps := allDeps.toArray.filter (· != theoremName)

    if filteredDeps.isEmpty then
      logInfo m!"[find_all_deps] '{theoremName}' has no theorem dependencies"
    else
      logInfo m!"[find_all_deps] All dependencies of '{theoremName}' ({filteredDeps.size} theorems):"
      for dep in filteredDeps.qsort Name.lt do
        logInfo m!"  {dep}"

@[command_elab findDepsDepth]
def elabFindDepsDepth : CommandElab := fun stx => do
  let theoremNameStr := stx[1].isStrLit?.get!
  let theoremName := theoremNameStr.toName
  let depthLit := stx[2]
  let depth := depthLit.isNatLit?.get!

  let env ← getEnv

  -- 检查定理是否存在
  match env.find? theoremName with
  | none =>
    logError m!"Theorem '{theoremName}' not found"
    return
  | some constInfo =>
    if !isTheorem env theoremName then
      logError m!"'{theoremName}' is not a theorem"
      return

    let depGraph := getDependenciesByLayers env theoremName depth
    let totalCount := depGraph.layers.foldl (fun acc layer => acc + layer.theorems.size) 0

    logInfo m!"[find_deps_depth] Dependencies of '{theoremName}' within depth {depth} ({totalCount} theorems):"

    for layer in depGraph.layers do
      if layer.theorems.size > 0 then
        logInfo m!"  Layer {layer.depth} ({layer.theorems.size} theorems):"
        for thm in layer.theorems.qsort Name.lt do
          if thm != theoremName then  -- 不显示根定理本身
            -- 显示该定理的直接依赖
            match depGraph.edges.get? thm with
            | some deps =>
              if deps.size > 0 then
                let depsStr := deps.foldl (fun acc dep => acc ++ s!" {dep}") ""
                logInfo m!"    {thm} → depends on:{depsStr}"
              else
                logInfo m!"    {thm}"
            | none => logInfo m!"    {thm}"

@[command_elab exportDepsJson]
def elabExportDepsJson : CommandElab := fun stx => do
  let theoremNameStr := stx[1].isStrLit?.get!
  let theoremName := theoremNameStr.toName
  let depthLit := stx[2]
  let depth := depthLit.isNatLit?.get!
  let filename := stx[3].isStrLit?.get!

  let env ← getEnv

  -- 检查定理是否存在
  match env.find? theoremName with
  | none =>
    logError m!"Theorem '{theoremName}' not found"
    return
  | some constInfo =>
    if !isTheorem env theoremName then
      logError m!"'{theoremName}' is not a theorem"
      return

    let depGraph := getDependenciesByLayers env theoremName depth

    -- 构建边的JSON表示
    let edgesList := depGraph.edges.fold (fun acc thmName deps =>
      (thmName.toString, Json.arr (deps.map (Json.str ∘ Name.toString))) :: acc) []

    let jsonData := Json.mkObj [
      ("root_theorem", Json.str theoremName.toString),
      ("max_depth", Json.num depth),
      ("layers", Json.arr (depGraph.layers.map fun layer =>
        Json.mkObj [
          ("depth", Json.num layer.depth),
          ("theorems", Json.arr (layer.theorems.map (Json.str ∘ Name.toString)))
        ])),
      ("edges", Json.mkObj edgesList)
    ]

    -- 写入文件
    IO.FS.writeFile filename (toString jsonData)
    logInfo m!"[export_deps_json] Exported dependency graph to '{filename}'"

-- 测试用例（可以直接在这个文件中使用）
#check Nat.lt_trichotomy
#find_deps "Nat.lt_trichotomy"
#find_deps_depth "Nat.lt_trichotomy" 2
#find_deps_depth "Nat.lt_trichotomy" 3
#export_deps_json "Nat.lt_trichotomy" 3 "nat_lt_trichotomy_deps.json"
#find_all_deps "Nat.lt_trichotomy"

```



## 使用方法

添加一份lean文件在项目下，需要使用到该工具的时import这份文件，个人经验是需要注意在import Mathlib之前。