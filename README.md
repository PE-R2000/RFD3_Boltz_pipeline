# RFD3_Boltz_pipeline
## Practical Guide to the Controls in This Notebook



This appendix explains what the main controls do, when to change them, and which settings are best left alone unless you are deliberately reproducing a protocol.



The workflow in this notebook has four stages:



1. Load a starting structure.

2. Generate backbone designs with RFdiffusion3 (RFD3).

3. Design amino-acid sequences with ProteinMPNN or LigandMPNN.

4. Optionally run Boltz-2 to predict and rank the final designs.



If you are new to this workflow, change only a few settings at a time. A good first run uses one structure, one batch, a small `diffusion_batch_size`, default advanced settings, and a small Boltz sample count.



> **Known notebook caveats**

>

> - The main workflow is usable, but a few advanced notebook features still need cleanup.

> - `generate_yamls_only` is exposed in the interface, but it is not yet fully wired to skip Boltz prediction.

> - Upload mode and Boltz YAML export should be treated as features that may need validation if you rely on them heavily.

> - The output path note earlier in the notebook places MPNN outputs under `batch_XX`, but the current implementation writes them under `outputs/{design_name}/MPNN_outputs/`.



## Start Here: Which Settings Matter For My Task?



Use this quick guide before diving into the full parameter reference.



- **Small-molecule binder design**: focus on `ligand`, `length`, `select_fixed_atoms`, `select_buried`, and `select_exposed`.

- **Protein binder design**: focus on `contig`, `select_hotspots`, `infer_ori_strategy`, and `is_non_loopy`.

- **Enzyme or active-site scaffolding**: focus on `unindex`, `select_fixed_atoms`, `length`, and sometimes `redesign_motif_sidechains`.

- **DNA or RNA binder design**: focus on `contig`, `ori_token`, `select_fixed_atoms`, and optional hydrogen-bond constraints.

- **Symmetric assemblies**: focus on `symmetry_id`, `is_unsym_motif`, `is_symmetric_motif`, and `diffusion_kind="symmetry"`.

- **Redesigning an existing fold instead of starting from scratch**: focus on `partial_t` and a careful definition of which atoms stay fixed.



## Safe Defaults For First Runs



These defaults are good starting points for many exploratory runs.



- Keep `num_batches = 1` while debugging inputs.

- Keep `diffusion_batch_size` small, such as `1` to `4`.

- Leave most advanced diffusion settings unchanged unless you are reproducing a known recipe.

- Keep `prevalidate_inputs = True` so bad selections fail early.

- Keep `is_non_loopy = True` unless you want more flexible or loop-heavy backbones.

- Keep `low_memory_mode = False` unless memory is tight.

- For Boltz, start with a small `boltz_diffusion_samples` value such as `1` to `3`.



## The Most Important Idea: How You Describe The Design



RFD3 needs a description of what to copy from the input structure and what to create from scratch. The most important controls for that are `length`, `contig`, `unindex`, `ligand`, and the selection fields.



### `length`



`length` tells RFD3 how long the designed system should be.



- Use a single value such as `180` when you want one exact total length.

- Use a range such as `180-220` when you want RFD3 to explore several lengths.

- In nucleic-acid or multi-component cases, this is the total polymer length, not just the length of the newly designed protein chain.

- Do **not** use `length` together with `partial_t`; partial diffusion uses the input structure length.



### `contig`



`contig` is the main composition string. It tells RFD3 which parts come from the input structure and which parts should be generated.



Rules of thumb:



- A residue with a chain label such as `A10-25` means "use these residues from the input structure".

- A number or range without a chain label such as `50` or `40-60` means "design a new segment of this length".

- `/0` means a chain break.

- Commas separate pieces of the design.



Examples:



- `A10-25,50-80` means "copy residues A10 to A25, then design a connected segment that is 50 to 80 residues long".

- `40-60,/0,E6-155` means "design a new chain of 40 to 60 residues, then keep target residues E6 to E155 as a separate chain".

- `A1-10,/0,B15-24,/0,120-130` means "keep two nucleic-acid segments as fixed chains and design a separate protein chain of 120 to 130 residues".



### `unindex`



`unindex` is for motif residues that matter chemically, but whose exact sequence position is not fixed in advance.



This is especially useful in enzyme design or active-site scaffolding.



In plain language: use `unindex` when you know certain residues or ligand-contact atoms must survive, but you do not want to force them to stay at one exact residue number in the final design.



Example:



- `A108,A139,A152,A156` means these motif pieces should be present, but the model can infer where they belong in sequence.



### `ligand`



`ligand` tells RFD3 which non-protein component from the input structure should be treated as important.



- Usually this is a three-letter CCD name such as `IAI`.

- Multiple ligands can be given as a comma-separated string such as `NAI,ACT`.

- Use this whenever the design should respond to a ligand, cofactor, metal, or other non-protein context.



## How Selections Work



Several fields use the same idea: you select residues or atoms and tell the model what role they should play.



These fields include:



- `select_fixed_atoms`

- `select_unfixed_sequence`

- `select_buried`

- `select_partially_buried`

- `select_exposed`

- `select_hbond_acceptor`

- `select_hbond_donor`

- `select_hotspots`



A selection can often be written in more than one way.



- A simple contig-like string such as `A10-25,B3-7`

- A dictionary such as `{"IAI": "ALL", "A108": "N,CA,C,O"}`

- In some cases a boolean such as `True` or `False`



### `select_fixed_atoms`



This is one of the most important controls.



In plain language: it says which atoms should stay in the same 3D position instead of being moved by the model.



Common patterns:



- `True`: fix all atoms pulled from the input.

- `False`: allow all pulled atoms to move.

- `{"IAI": "ALL"}`: keep all atoms of ligand `IAI` fixed.

- `{"A108": "N,CA,C,O"}`: keep only specific backbone atoms fixed.

- `BKBN`: shorthand for backbone atoms.

- `TIP`: shorthand for tip atoms used in motif-style conditioning.

- `ALL`: all atoms in that selected residue or component.

- `""` as the atom list: intentionally select none of the atoms for fixing.



### `select_unfixed_sequence`



This says where the amino-acid identity is allowed to change.



In plain language: fixed coordinates and fixed sequence are different things. A residue can stay in place in 3D while still being allowed to mutate.



Example:



- `A20-35` means residues A20 to A35 can change amino-acid identity.



### `select_buried`, `select_partially_buried`, `select_exposed`



These control surface accessibility.



In plain language:



- `select_buried`: atoms or residues that should end up tucked into the design.

- `select_exposed`: atoms or residues that should stay accessible to solvent.

- `select_partially_buried`: intermediate accessibility.



These are especially useful for small-molecule binding, where one face of the ligand should be buried and another face should remain exposed.



### `select_hbond_acceptor` and `select_hbond_donor`



These add atom-level hydrogen-bond conditioning.



Use them when you want the design to respect a known hydrogen-bond pattern, especially in nucleic-acid or catalytic designs.



Example:



- `{"C16": "N7,O6"}` marks those atoms as hydrogen-bond acceptors.



### `select_hotspots`



This marks residues or atoms on the target that the new design should contact.



In plain language: these are the places you most want the binder to engage.



This is one of the most important controls for protein binder design.



## RFD3 Basic Design Settings



These are the fields in the "RFD3 basic design settings" cell.



### `design_name`



This is only a run label.



- It determines output folder names and archive names.

- It does **not** affect the science or model behavior.



### `seed`



This controls random-number generation.



- Reusing the same seed with the same settings can help reproducibility.

- Changing the seed can give different designs even when everything else is unchanged.



### `diffusion_batch_size`



This is how many RFD3 designs are generated in one sampling batch.



- Larger values produce more structures per run.

- Larger values also use more memory.

- For symmetry jobs, small values are strongly recommended.



### `num_batches`



This repeats the RFD3 generation process multiple times.



- Total number of backbone designs is roughly `num_batches × diffusion_batch_size`.

- Use `1` while testing inputs.



### `redesign_motif_sidechains`



Use this when you want motif side chains to be redesigned instead of copied exactly.



This is useful when a motif backbone must stay but side-chain identities should be reconsidered.



### `plddt_enhanced`



This enables a confidence-related enhancement used by the model.



For most users: leave this on.



### `is_non_loopy`



This encourages more structured designs with fewer floppy loops.



- Often recommended for binder design.

- If you want more flexible or loop-rich geometries, you may choose to turn it off.



### `partial_t`



This turns the run into **partial diffusion** instead of fully de novo generation.



In plain language: the model starts from a known structure and perturbs it by a chosen amount, instead of inventing a backbone from scratch.



Practical guidance:



- Typical useful values are around `5.0` to `15.0` Å.

- Smaller values stay closer to the starting structure.

- Larger values allow more structural change.

- Do not combine this with `length`.



### `skip_existing`



If outputs already exist, this can skip rerunning the same designs.



This is useful for resuming interrupted work, but it can also confuse you if you forget it is on and expect new outputs.



### `dump_trajectories`



This saves intermediate diffusion trajectory files.



Useful for debugging or visualization, but usually not needed for routine runs.



### `align_trajectory_structures`



This aligns saved trajectory structures for easier inspection.



Usually leave this off unless you are studying the trajectory itself.



### `dump_prediction_metadata_json`



This saves metadata JSON files alongside the structures.



Useful for traceability and debugging. Usually leave it on.



### `output_full_json`



This controls how much JSON metadata is saved.



Usually leave it on unless file size becomes a concern.



### `prevalidate_inputs`



This checks the design specification before a long run begins.



For most users: keep this on.



### `low_memory_mode`



This switches RFD3 into a more memory-efficient mode.



- Use it when GPU memory is insufficient.

- Expect slower runs.



## RFD3 Advanced Conditioning Settings



These fields change the behavior of the design task itself.



### `infer_ori_strategy`



This tells RFD3 how to place the designed region relative to the input structure.



Options:



- `None`: do not infer an origin automatically.

- `com`: use the center of mass of the input structure.

- `hotspots`: infer placement from the hotspot region.



For protein binder design, the repository docs specifically recommend `hotspots`.



### `ori_token`



This is an explicit `[x, y, z]` position for the design origin.



In plain language: it helps steer where the new design is placed in 3D space relative to the input.



Use it when you know the approximate desired location of the new designed region.



### Symmetry Controls



These fields are only important for symmetric designs.



- `symmetry_id`: the symmetry group, such as `C2`, `C3`, `D2`, or `D4`.

- `is_unsym_motif`: motif or component names that should **not** be symmetrized, such as DNA strands or a single ligand.

- `is_symmetric_motif`: whether the input motif is already symmetric.

- `diffusion_kind`: use `symmetry` for symmetry-aware diffusion.



Practical guidance:



- Use symmetry only when the design task truly requires it.

- Start with `diffusion_batch_size = 1` for symmetry jobs.

- Expect higher memory use.



## Advanced Diffusion Sampler Settings



These controls affect how the diffusion process behaves. Most users should leave them at their defaults unless reproducing a known protocol.



### Usually understandable and sometimes useful



- `num_timesteps`: number of diffusion steps. More steps can increase runtime.

- `noise_scale`: how much random noise is injected during sampling. Lower values usually make results more conservative.

- `step_scale`: scales the step size during sampling. The docs note that larger values can give less diverse but more designable outputs.

- `gamma_0`: influences diversity. Lower values are often described as giving lower-temperature, more designable outputs.

- `center_option`: how coordinates are centered during inference. Options are `all`, `motif`, and `diffuse`.

- `s_trans`: translational augmentation scale.

- `s_jitter_origin`: how much random jitter to add to the origin.

- `allow_realignment`: whether motif-based realignment is allowed during sampling.



### Expert-only controls



These are best left unchanged unless you know exactly why you are changing them.



- `min_t`

- `max_t`

- `sigma_data`

- `s_min`

- `s_max`

- `p`

- `gamma_min`

- `fraction_of_steps_to_fix_motif`

- `skip_few_diffusion_steps`

- `insert_motif_at_end`

- `solver`



### Classifier-Free Guidance Controls



These are advanced steering options.



- `use_classifier_free_guidance`: turn guidance on.

- `cfg_scale`: how strongly guidance influences sampling.

- `cfg_t_max`: the latest timepoint up to which guidance is applied.



For most users: leave guidance off unless you are explicitly experimenting with conditional steering.



### Debug-Oriented Output Controls



These are useful mainly when diagnosing tricky setups.



- `cleanup_guideposts`: if turned off, keeps extra guidepost information that can help debug unindexed motif setups.

- `cleanup_virtual_atoms`: if turned off, keeps virtual atoms used internally during diffusion. This is mainly for debugging and may make downstream tools less happy.

- `read_sequence_from_sequence_head`: whether saved sequence annotations come from the model sequence head. Usually leave this on.

- `global_prefix`: adds a custom filename prefix to outputs.



## Sequence Design With ProteinMPNN Or LigandMPNN



This stage takes each RFD3 backbone and proposes amino-acid sequences that fit it.



### `mpnn_model_type`



Choose the sequence-design model.



- `protein_mpnn`: use this for protein-only contexts.

- `ligand_mpnn`: use this when ligands or non-protein context matter.



If your design contains a small molecule, cofactor, or similar context, `ligand_mpnn` is usually the better choice.



### `mpnn_batch_size` and `mpnn_number_of_batches`



These determine how many sequence proposals are generated.



- `mpnn_batch_size`: number of samples drawn per batch.

- `mpnn_number_of_batches`: how many such batches to run.



More samples mean more diversity, but also more outputs to inspect.



### `mpnn_temperature`



This controls sampling diversity.



- Lower temperature gives more conservative sequences.

- Higher temperature gives more diverse and sometimes riskier sequences.



### `mpnn_seed`



Controls reproducibility for sequence sampling.



### `mpnn_remove_waters` and `mpnn_remove_ccds`



These clean the structure context before sequence design.



- `mpnn_remove_waters = True` is often sensible.

- `mpnn_remove_ccds` removes named chemical components you do not want considered.



### `mpnn_structure_noise`



Adds coordinate noise in Å before sequence design.



Most users should leave this at `0.0` unless deliberately testing robustness.



### `mpnn_decode_type`



This controls how residues are decoded.



- `auto_regressive`: each prediction uses previously predicted residues. This is the standard inference setting.

- `teacher_forcing`: each prediction sees the original input sequence at previous positions.



For most users: keep `auto_regressive`.



### `mpnn_causality_pattern`



This controls what each residue is allowed to "see" during decoding.



Options include:



- `auto_regressive`

- `unconditional`

- `conditional`

- `conditional_minus_self`



This is an expert control. Keep the default unless you know why another pattern is needed.



### `mpnn_initialize_with_ground_truth`



This decides whether sequence embeddings start from the known input sequence or from an unknown/blank state.



For ordinary design runs, the default is usually appropriate.



### `mpnn_atomize_side_chains`



Relevant mainly for `ligand_mpnn` and advanced fixed-residue contexts.



Most users can leave this off.



### Defining What MPNN May Change



These fields define the design scope.



- `mpnn_fixed_residues`: exact residues that must not change.

- `mpnn_designed_residues`: exact residues that are allowed to change.

- `mpnn_fixed_chains`: whole chains that must not change.

- `mpnn_designed_chains`: whole chains that are allowed to change.



Use only one of these approaches at a time. They are different ways to say what is mutable.



### Amino-Acid Preferences And Restrictions



These controls let you push the sequence sampler toward or away from specific residue identities.



- `mpnn_bias`: global amino-acid preferences.

- `mpnn_bias_per_residue`: residue-specific amino-acid preferences.

- `mpnn_omit`: amino acids to forbid globally.

- `mpnn_omit_per_residue`: amino acids to forbid at specific residues.

- `mpnn_pair_bias`: pairwise amino-acid preferences.

- `mpnn_pair_bias_per_residue_pair`: residue-pair-specific preferences.



These are powerful but advanced. Use them only when you already know the sequence chemistry you want to encourage or avoid.



### `mpnn_temperature_per_residue`



This lets you vary sampling temperature across positions.



Useful for expert workflows, but not usually needed for first-pass design.



### MPNN Symmetry Controls



These are separate from RFD3 structural symmetry.



- `mpnn_symmetry_residues`: groups of equivalent residues that should be treated symmetrically.

- `mpnn_symmetry_residues_weights`: weights for those symmetry groups.

- `mpnn_homo_oligomer_chains`: treat matching positions across specified chains as equivalent.



Use these when sequence symmetry matters even after backbone generation.



## Boltz-2 Prediction And Ranking Controls



This stage is optional. It uses Boltz-2 as an external prediction engine after sequence design.



In plain language: RFD3 proposes structures, MPNN proposes sequences, and Boltz provides another layer of structural prediction and ranking.



### `boltz_diffusion_samples`



How many Boltz prediction samples to run for each MPNN structure.



- More samples can improve coverage.

- More samples also increase runtime.



### `boltz_seed`



Reproducibility control for Boltz runs.



### `boltz_affinity`



Whether affinity-related properties should be requested and used in ranking.



If turned on, the notebook also tries to incorporate affinity information into the final ranking table.



### `boltz_ligand_chain`



This tells the notebook which chain should be treated as the ligand or binder chain for affinity and interface metrics.



If this chain is wrong, the ranking metrics for burial and contacts may be misleading.



### `boltz_use_msa_server`



Whether to use an external MSA server when preparing Boltz inputs.



### `generate_yamls_only`



Intended meaning: prepare Boltz YAML files without running prediction.



Current status: this control is exposed in the notebook interface, but the implementation should be treated as incomplete and validated carefully before relying on it.



### How The Ranking Works



The ranking table combines several signals into a single `BoltzGen_Score`.



The current notebook uses:



- prediction confidence

- affinity probability

- predicted affinity value

- interface burial via `delta_sasa`

- polar contact counts

- a simple sequence liability score



This is a **heuristic ranking**, not proof that a design will bind or behave well experimentally.



## Output Locations



The notebook writes outputs under the Colab root.



Current implementation paths:



- `/content/outputs/{design_name}/batch_XX/RFD3_designs/` for RFD3 backbone designs

- `/content/outputs/{design_name}/MPNN_outputs/` for MPNN-designed sequences and CIFs

- `/content/outputs/{design_name}/boltz/` for Boltz YAML files and prediction outputs

- `/content/outputs/{design_name}/ranking/boltz_ranking.xlsx` for the final ranking workbook

- `/content/downloads/{design_name}_results.zip` for the packaged ZIP archive



## Common Mistakes And How To Recover



### The run fails immediately with a validation error



Usually this means one of the following:



- a selection string does not match the input structure

- a dictionary selection is malformed

- `partial_t` was used together with `length`

- the input structure was provided but not actually used by `contig`, `unindex`, `ligand`, or `partial_t`



### The model runs, but the outputs are not changing



Check:



- whether `skip_existing = True` reused earlier outputs

- whether you changed only notebook display settings instead of model settings

- whether the same `seed` and the same design settings were reused



### The design does not appear near the intended binding site



Check:



- `select_hotspots`

- `infer_ori_strategy`

- `ori_token`

- whether your target residues are correctly named and selected



### The symmetry job runs out of memory



Try:



- `diffusion_batch_size = 1`

- `low_memory_mode = True`

- fewer concurrent designs



### MPNN gives unrealistic sequence proposals



Try:



- lowering `mpnn_temperature`

- removing unnecessary advanced bias settings

- verifying that the correct model type was chosen

- checking whether the structure still contains unwanted context components



### The Boltz ranking looks strange



Check:



- whether `boltz_ligand_chain` points to the intended binder or ligand chain

- whether Boltz was properly installed and the runtime restarted after installation

- whether you are interpreting `BoltzGen_Score` as a heuristic rather than a guarantee



## Practical Advice



A good experimental workflow is:



1. Make the smallest possible first run.

2. Confirm the geometry is sensible before sampling many sequences.

3. Increase sequence sampling only after the backbone task is behaving as expected.

4. Use Boltz ranking after you already trust the backbone and sequence-design setup.

5. Keep notes on which settings changed between runs.



If you are unsure which field to change, start with `contig`, `length`, `ligand`, and the simplest selection dictionary needed to express your design idea. Leave the diffusion math controls alone unless you are reproducing a published or previously validated protocol.
