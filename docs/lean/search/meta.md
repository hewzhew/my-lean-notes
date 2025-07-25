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

## 使用方法

添加一份lean文件在项目下，需要使用到该工具的时import这份文件，个人经验是需要注意在import Mathlib之前。
