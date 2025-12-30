## Create network
```
openfe plan-rbfe-network -M fxr_ligand.sdf\
 -p fxr_prot.pdb\
 -o network_setup\
 --n-protocol-repeats 2 -s setting.yaml
```

## Perform Simulation
```run simulation.sh
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
  cmd="openfe quickrun \"$file\" -o results/ -d results/1/$dirpath"

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

## Post-simulation processing
- fxr_agonist_openfe_rbfe.ipynb
