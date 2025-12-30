## Create network
A star network was constructed with compound FXR_12 serving as the central node. Partial charges were calculated using the Neural Atomistic Graph Learning (NAGL) method, and each edge in the constructed star network was repeated twice to ensure the reliability of subsequent topological analysis.

```
openfe plan-rbfe-network -M fxr_ligand.sdf\
 -p fxr_prot.pdb\
 -o network_setup\
 --n-protocol-repeats 2 -s setting.yaml
```

## Perform Simulation

Automatically schedule each edge's calculation task (JSON) with SLURM and output results to the results directory. Let's have a shell script as follows;

```run_simulation.sh
#!/bin/bash

for file in network_setup/transformations/*.json; do
  # 提取相对路径
  relpath="${file#network_setup/transformations/}"
  dirpath="${relpath%.json}"

  # 提取 phase（最后一个下划线后的内容）
  phase="${dirpath##*_}"

  # 根据 phase 确定后缀
  case "$phase" in
    complex) suffix="C" ;;
    solvent) suffix="S" ;;
    *)       suffix="X" ;;  # 未知情况兜底
  esac

  # 去掉末尾的 _phase，得到 ..._FXR_10
  temp="${dirpath%_$phase}"
  # 提取目标编号（最后一个 _ 后的数字）
  target_num="${temp##*_}"

  job_name="RBFE_${target_num}${suffix}"
  jobpath="network_setup/transformations/${dirpath}.job"
  cmd="openfe quickrun \"$file\" -o results/ -d results/"

  # 生成 Slurm 脚本
  {
    printf '%s\n' '#!/usr/bin/env bash'
    printf '#SBATCH --job-name=%s\n' "$job_name"
    printf '#SBATCH --cpus-per-task=4\n'
    printf '#SBATCH --gres=gpu:1\n'
    printf '#SBATCH --time=720:00:00\n'
    printf '\n%s\n' "$cmd"
  } > "$jobpath"

  sbatch "$jobpath"
done
```
Submit job with the command:
```
sh run_simulation.sh
```
Simply wait for all calculations to finish. For each complex (run in duplicate), the required computation time is about 7 hours on an RTX 4090 GPU.

## Collect results and generate reports
After completing the simulation steps, you can gather all calculation results and automatically generate a DDG report with one command. Execute the following command in the terminal, where ./openfe_results refers to the directory storing your OpenFE calculation results, and the --report ddg parameter specifies to generate a DDG (relative binding free energy) report:
```
openfe gather --report ddg ./openfe_results
```
The generated DDG report (unit: kcal/mol) is as follows, including the ligand pairs, relative binding free energy value DDG(i->j) and corresponding uncertainty for each pair:
```
┌──────────┬──────────┬──────────────────────┬────────────────────────┐
│ ligand_i │ ligand_j │ DDG(i->j) (kcal/mol) │ uncertainty (kcal/mol) │
├──────────┼──────────┼──────────────────────┼────────────────────────┤
│ FXR_12   │ FXR_10   │ 2.00                 │ 0.04                   │
│ FXR_12   │ FXR_74   │ -0.308               │ 0.009                  │
│ FXR_12   │ FXR_76   │ 1.5                  │ 0.2                    │
│ FXR_12   │ FXR_77   │ -1.4                 │ 0.2                    │
│ FXR_12   │ FXR_78   │ -1.5                 │ 0.3                    │
│ FXR_12   │ FXR_79   │ 3.0                  │ 1.0                    │
│ FXR_12   │ FXR_81   │ -1.2                 │ 0.1                    │
│ FXR_12   │ FXR_82   │ -0.3                 │ 0.2                    │
│ FXR_12   │ FXR_83   │ -2.0                 │ 0.4                    │
│ FXR_12   │ FXR_84   │ 0.66                 │ 0.08                   │
│ FXR_12   │ FXR_85   │ -0.28                │ 0.07                   │
│ FXR_12   │ FXR_88   │ -0.1                 │ 0.5                    │
│ FXR_12   │ FXR_89   │ 0.8                  │ 0.2                    │
└──────────┴──────────┴──────────────────────┴────────────────────────┘
```


## Post-simulation processing

Upon the completion of all simulation jobs, carry out post-simulation data processing and analysis using the Jupyter Notebook: fxr_agonist_openfe_rbfe.ipynb.
