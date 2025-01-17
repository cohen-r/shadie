
# SHaDiE
## "Simulating Haploid-Diploid Evolution"

### Description of project goal:
`shadie` will be a wrapper around the forward-in-time evolutionary simulation program [SLiM3](https://messerlab.org/slim/). It will accept a phylogeny from the user and generate equivalent population demography inside a script that can be fed directly into SLiM3. THe user will also be able to define a number of different parameters for the simulation and the program will prepare the slim script appropriately along with a dispatch script, which is necessary to run SLiM3 from the command line. The user will be able to run the SLiM simulation using a `shadie` command. After the simulation is complete, `shadie` will also provide a number of summary statistics and methods for inspecting the output of the simulation. 

`shadie` is specifically designed to model the varied biphasic lifecycles in plants, allowing the user to explore the evolutionary consequences of different life histories. 


### Description of the code:
Possible dependencies:
* `numpy`: to manipulate data
* `pandas`: to organize and analyze data.
* `toytree`: to generate phylogenies.
* `msprime`: process and read .tree objects output by SLiM3
* `pyslim`: process and read .tree objects output by SLiM3
* `tskit`: used by `pyslim`


Classes:
- `Script`: the script file that will be generated by the package to run SLiM3
- `Demography`: creates subpopulation demoography for SLiM based on phylogeny input by user
- `Stats`: generates various summary statistics

Parameters (User-controlled):
- `lifehistory`:selection will act differently on haploid and diploid life stages depending on the kind of lineage the user wants to model
- `genome`: the size of the haploid set of genetic material upon which evolution is being simulated
- `genelement`: types of genomic elements, e.g. introns, exons, non-coding regions
- `muttype`: defines allowable mutation types in the simulation
- `chromosome`: allows user to make adjustments to linear arrangement of genomes into genes and other regions
- `tree`: phylogeny (newick or .tree object from toytree) that is used to generate subpopulation demography

Parameters (not editable by user):
- `dispatch`: the dispatch script generated by `Script` class; may include instructions for collecting summary statistics too
- `model`: SLiM3 can use Wright-Fisher (WF), extended WF, or non Wright-Fisher models
- `reproduction`: allows different mating models, e.g. clonal, slefing, hermaphroditic, sexual, and combinations thereof
- `fitness`: defines the fitness effects of different mutations
- `mutation`: changes to the genome


### Description of the data:
Users will be able to input a phylogeny and the program will generate the corresponding population demography for input into SLiM. 

The `Demography` submodule of `shadie` will take the phylogeny from the user (provided as newick string or toytree object). It will use `toytree` to traverse the tree from root to tips and collect information it needs to generate the SLiM demography in a `pandas` dataframe, for example:

| source    | child2    | nodeheight| gen       |
| --------- | --------- | --------- | --------- |
| 0         | 3         | 1000000   | 1         |
| 3         | 5         | 600000    | 801       |
| 0         | 2         | 600000    | 801       |
| 3         | 4         | 300000    | 1401      |
| 0         | 1         | 300000    | 1401      |
 
- each node moving from tips to root is renamed with the lowest number of the child tips
- `gen` = 1+(abs(nodeheight-treeheight)*(gentime/treeheight))
- This example data is from a tree generated in toytree, which has no outgroup. Because the first split happens at gen1, we probably want a burn-in time before the simulation begins. 

 This will be used to generate this part of the script, which controls when populations in the program split into subpopulations:

```Java
#write beginning row:
1 { sim.addSubpop("p1", K); 
}
#for row in pandas DF, write the following:
{gen}{
    sim.addSubpopSplit("p{dest}", K, p{source});
    p{source}.setMigrationRates(p{dest}, 1.0);
}
{gen+1}{
    p1.setMigrationRates(p2, 0.0);
}
#append until out of rows

#then, write last line:
self.gentime late() { 
    sim.outputFull(); 
}
```

Beyond that, `shadie` will largely generate its own data and output summary statistics that can be used to evaluate observed data. The user will choose from a relatively simple set of options.


### Demonstration of user interaction
The user can optionally provide a phylogeny, which will control the demography of the simulation (detailed above).

User can then adjust other simulation parameters, such as life history, mutation rate, and recombination rate. `package` has built-in defaults for many values (which are adjusted dynamically by the script based on the parameters the user *does* define), so the user does not have to define every parameter for every run:
```python
# example of user-defined inputs and generated slim script file:

#define simulation parameters
sim1 = Shadie(
	Ne = 1000, 
	organism="pter",  
	mutrate=1e-7,
	recomb=1e-9,
	genome_size=1e6)

sim1.write() #writes script
...
```

Once the user is finished preparing the simulation script they can run a simulation in SLiM3:
```python
sim1_out = sim1.runSLIM #feeds script into SLiM3
```

User can calculate dN/dS, perform an MK-test, visualize mutations on a phylogeny, etc...
```python
# example of summary statistics from SLiM output
#MK-test
sim1_out.mk

#dN/dS
sim1_out.dnds

#draw gene trees?
sim1out.draw
...
```

### Similar Tools
[`pyslim`](https://github.com/tskit-dev/pyslim) is a python module that handles tree sequence files output by SLiM as a thin interface with `tskit`. This module will make it much easier to generate summary statistics for the user, as they've already done a lot of the work. `pyslim` will also allow `program` to use a combination of SLiM forward-in-time simulation and coalescent in `msprime` to run simulations more quickly. 

To my knowledge, there are no other publicly available python wrappers for SLiM3 and no other evolution simulation programs that were built to model selection on biphasic lifecycles. 
