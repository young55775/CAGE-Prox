# Computer Aided and Genetically Encoded PROXimal decaging (CAGE-Prox)

## 1. Structure Preparation

### 1.0 Building Parameter File for Ligand Molecule

Save the protein and ligand into seperate pdb files:

    grep ^ATOM complex.pdb > pro.pdb
    grep ^HETATM complex.pdb > lig.pdb

Add hydrogens using other softwares like **Avogadro**, and save as mol2 format. Then generate Rosetta params (Optional: you can use **acpype.py** to generate more accurate charge based on **ambertools**)

    acpype.py -i lig.mol2 -n [charge]
    convert_qm2mol2.sh lig_bcc_gaff.mol2
    /path/of/rosetta/source/scripts/python/public/molfile_to_params.py \
      -n LIG \
      --extra_torsion_output lig_bcc_gaff.mol2

### 1.1 Relax in Torsion Space
Put the protein and ligand structure into a single pdb, and **relax** in internal coordinates:

    cat pro.pdb LIG_0001.pdb > complex.pdb
    /path/of/rosetta/source/bin/relax.mpi.linuxgccrelease \
      -s complex.pdb \
      -extra_res_fa LIG.params \
      -relax:constrain_relax_to_start_coords \
      -ramp_constraints false \
      -relax:coord_constrain_sidechains \
      -nstruct 40 \
      -ex1 \
      -ex2 \
      -use_input_sc \
      -flip_HNQ \
      -no_optH false

### 1.2 Relax in Cartesian Space
Select the structure with the lowest score, fix the ligand with a **movemap** (not necessary):

    RESIDUE * BBCHI
    RESIDUE [number of ligand] NO

Relax the structure in Cartesian coordinates:

    /path/of/rosetta/source/bin/relax.mpi.linuxgccrelease \
      -s complex.pdb \
      -extra_res_fa LIG.params \
      -relax:constrain_relax_to_start_coords \
      -ramp_constraints true -relax:coord_constrain_sidechains \
      -nstruct 200 \
      -ex1 \
      -ex2 \
      -use_input_sc \
      -flip_HNQ \
      -no_optH false \
      -relax:cartesian \
      -score:weights talaris2014_cart \
      -in:file:movemap movemap \
      -crystal_refine

## 2. Pocket Localization
We define those residues proximal to the known ligand position (add correct direction) as pocket residues.

    /path/of/rosetta/source/bin/gen_lig_grids.mpi.linuxgccrelease \
      -s pro.pdb lig.pdb \
      -grid_active_res_cutoff 6.0 \
      -vector_angle_threshold 90 \
      -ignore_zero_occupancy false \
      -ignore_unrecognized_res > log

This app will generate pos file and grid file, we only need pos file which has all proximal residues' index, raw distance and angle information can be found in log file.

## 3. Geometry Filter
Using the pos file created by **gen_lig_grids**, we can now generate **mutfile** for each residue that passes the geometry filter. The mutfile is defined as

    total 1
    1
    L 278 Y

The first line starts with **total** and the number of all mutations, in this case we only have one mutation for one residue so it is always **1**. The following block defines one round of calculation which mutates L278 to Y.

First we need to map the pdb residue number to its index

    /path/of/rosetta/source/src/python/apps/public/pdb2fasta.py pro.pdb > pro.fasta
    get_seq_list.sh pro.fasta > seq.pro.txt

Then, generate single mutation information line by line

    pos_to_mutlist.py seq.pro.txt pro.pdb_0.pos > mut.list

Finally, create mutfile and a list

    mkdir mutfiles
    gen_mutfile.py mut.list
    ls mutfiles/*.mutfile > mutfile.list

## 4. DDG Calculation
All the command lines can be generated by

    run_mutations_ddg.py pro_0001.pdb mutfile.list \
      [cart_ddg path] \
      [-extra_res_fa LIG.params] \
      [-score:extra_improper_file LIG.tors] > job.list

The path of command and extra parameter files can be added by options or minor modification in the python script.

Then submit the job.list or simply run

    cat job.list | parallel -j [nproc]

## 5. Parsing Results
The DDG calculation will generate a **ddg** file for each mutfile. We can parse all the results by

    ls *ddg > list
    parse_data_ddg.py > results.txt

There are six column for each mutation

    column1: mutation
    column2: ΔG_fold
    column3: ΔG_bound
    column4: ΔΔG
    column5: G_bound
    column6: G_unbound

Final suggestion list can be selected from the results.
