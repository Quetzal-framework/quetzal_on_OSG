# Quetzal_on_OSG

iDDC modeling and inference using the Quetzal framework on Open Science Grid
for distributed High Throughput Computing (dHTC).

> :bulb: The [OSG](https://opensciencegrid.org/) is a consortium of research collaborations, campuses, national
> laboratories and software providers dedicated to the advancement of all open
> science via the practice of distributed High Throughput Computing (dHTC)

> :bulb: Quetzal is a suit of tools I developed for iDDC modeling, including:
- [Quetzal-CoaTL](https://github.com/Becheler/quetzal-CoaTL)
- [Quetzal-EGGS](https://github.com/Becheler/quetzal-EGGS)
- [Quetzal-CRUMBS](https://github.com/Becheler/quetzal-CRUMBS)
- [Quetzal-NEST](https://hub.docker.com/repository/docker/arnaudbecheler/quetzal-nest)

## What problem does this project solve?

:rocket: Running an iDDC inference workflow is an ambitious task. You need many different data,
many tools, a lot of computational fire power and a lot of scripting to glue all of this together.

 :crying_cat_face: That is not in the reach of many people interested in statistical phylogeography

:skull_and_crossbones: And have you even thought about reproducibility? :scream_cat:

:gift: Here we glue together all Quetzal tools to build a rather complicated workflow
and distribute its jobs across the Open Science Grid so you can run a spatially
explicit inference swiftly and without (too much) hassle!

:rainbow: Hopefully it will make your life easier and your iDDC goals more reachable.

## How to use

### 1- Understand the pipeline

:point_down::point_down::point_down: Have a look at the complete workflow illustrated at the end of this page!

### 2 - Replicate the example results

1. You need [an account on Open Science Grid](https://opensciencegrid.org/).
2. Then connect to your login node with e.g. `ssh user@login.osgconnect.net`
3. Clone this repository with `git clone https://github.com/Becheler/quetzal_on_OSG.git`
4. Enter the repo with `cd quetzal_on_OSG`
5. Initialize some symbolic links with `chmod u+x src/init.sh && ./src/init.sh`
6. Make an output directory and enter it: `mkdir output && cd output`
7. Run the example inference with 10 simulations, 2 repetitions in case of failure, and a SDM projected from the LGM (`-199`) to today (`20`):
     - `./../generate_DAG 10 2 $(seq -s ' ' -199 1 20) > flow.dag`
     - submit with `condor_submit_dag flow.dag`
     - assess the advancement with `condor_watch_q`
8. Once the simulations are done:
   - retrieve summary statistics: `sh src/post-analysis/get_sumstats.sh`
   - retrieve parameters table: `sh src/post-analysis/get_param_table.sh`
9. Use ABC-RF package (CRAN) to perform inference!

### 3 - Try change the user input

The following input files are defined by the user and should be placed in the `input_files` folder:

- `suitability.tif`: the suitability geotiff file for representing the landscape:
 - NA for representing ocean cells
 - 0 for representing continental cells of null suitability and infinite friction
 - ]0,1] values for intermediate suitability and frictions
- `EGG.conf`: configuration file for the quetzal-EGG used - unknown parameter values will be rewritten by the ABC pipeline
- `sample.csv`: file mapping sampled gene copies IDS to their latitude longitude sampling points
- `imap.txt`: IMAP file from BPP mapping individuals to putative populations for computing summary statistics

### 4 - Debugging :bug:

* Check progress of scheduled jobs with `condor_q` or `condor_watch_q`
* If a job fails, check `cat *out` to see if there is any executable path error.
* If a job is hold, something is wrong:
    - use `condor_q -nobatch` to get the ID of the job being on hold
    - use `condor_q -better-analyze <JOB-ID>` to check for bug reasons (generally wrong paths or files not found)

## The pipeline :see_no_evil:

- In green, the user inputs
- In blue, databases that are fetched or built
- In purple, the main output of this workflow

```mermaid
flowchart TD;
  start([START]) ---> margin & sample & species;

  classDef userInput fill:#6ad98b,stroke:#333,stroke-width:2px;
  classDef database fill:#66deff,stroke:#333,stroke-width:2px;
  classDef movie fill:#eb96eb,stroke:#333,stroke-width:2px;

  margin[/"margin buffer<br>(in degrees)"/]--infers---bbox[spatial extent];
  sample[/"sampling points<br>(shapefile)"/]--infers---bbox;
  species[/"species name"/]--->1-get-gbif.sh;

  class margin userInput;
  class sample userInput;
  class species userInput;

  bbox --> 3-get-chelsa.sh & 1-get-gbif.sh;

  subgraph 1-get-gbif.sh
      A[(GBIF)]-- crumbs.get_gbif -->B(occurrences.shp);
      class A database;
  end

  B--crumbs.animate-->2-visualize-gbif.sh;

  subgraph 2-visualize-gbif.sh
      C>occurences.mp4];
      class C movie;
  end

  subgraph 3-get-chelsa.sh
      D[(CHELSA)]-- crumbs.get_chelsa ---E[/"bioclimatic variables and times"/];
      class D database;
      class E userInput;
      E-. download -.-F1 & F2 & F3 & F4 & F5;

      subgraph world
          F1(1990);
          F2(...);
          F3(t);
          F4(...);
          F5("-21000");
      end

      subgraph landscape
          F1-. crop -.-FF1(1990);
          F2-. crop -.-FF2(...);
          F3-. crop -.-FF3(t);
          F4-. crop -.-FF4(...);
          F5-. crop -.-FF5("-21000");
      end

  end

  FF1------>G;

  subgraph "4-sdm.sh"

      subgraph classifiers
          H1((Random<br />Forest));
          H2((Extra<br />Trees));
          H3((XGB));
          H4((LGB));
      end

      B---->G(crumbs.sdm);
      G --fitting--- H1 & H2 & H3 & H4;
      H1 -- interpolation --> I1[pred4];
      H2 -- interpolation --> I2[pred3];
      H3 -- interpolation --> I3[pred2];
      H4 -- interpolation --> I4[pred1];

      I1 & I2 & I3 & I4 -- averaging ---K1;

      subgraph suitability
          subgraph present
              K1(1990);
          end
          subgraph past
              FF2-.-K2(...);
              FF3-.-K3(t);
              FF4-.-K4(...);
              FF5-.-K5("-21000");
          end
      end
      classifiers--extrapolation<br>and<br>averaging-->past;
      suitability--crumbs.to_geotiff-->suit-file[multiband suitability raster];

  end

  subgraph 5-visualize-sdm
      L>suitability.mp4];
      class L movie;
  end

  suit-file-- crumbs.animate --> 5-visualize-sdm;

  subgraph 6-gis
      suit-file-- crumbs.interpolat -->interpolated("interpolate missing bands along time axis<br>(1 band per generation)");
      interpolated-- crumbs.circle_mask --> circular(circular landscape)
      circular-- crumbs.rotate_and_rescale -->rotated-rescaled(Finer/Coarser rotated landscape);

  end

  subgraph 7-visualize-temporal-interpolation
      interpolated2>suitability-interpolated.mp4];
      class interpolated2 movie;
  end

  subgraph 8-visualize-circular-landscape
      circular2>suitability-circular.mp4];
      class circular2 movie;
  end

  subgraph 9-visualize-rotated-rescaled
      rotated2>suitability-rotated-rescaled.mp4];
      class rotated2 movie;
  end

  interpolated-- crumbs.animate --> 7-visualize-temporal-interpolation;
  circular-- crumbs.animate --> 8-visualize-circular-landscape;
  rotated-rescaled-- crumbs.animate --> 9-visualize-rotated-rescaled;

  rotated-rescaled --> EGG;

  subgraph 10-EGG
      EGG(Quetzal EGG);
      config[/configuration file/]-->EGG;
      class config userInput;
      hyperparameters[/hyperparameters/]--crumbs.sample-->params[parameters];
      class hyperparameters userInput;
      params-->EGG;
      sample2[/"sampling points<br>(shapefile)"/]-->EGG;
      class sample2 userInput;
      EGG-. n distributed lineages simulations .->output[(output.db)];
      class output database;
  end

  subgraph 11-post-treatment
      output-- crumbs.simulate_phylip_sequences<br>crumbs.phylip2arlequin-->pods[pseudo-observed data];
      pods--arlsumstats-->sumstats(summary statistics);
      output-- crumbs.retrieve_parameters -->paramtable(reference table);
      sumstats & paramtable --> ABCRF(ABC Random Forest);
  end
  ABCRF-->posterior>posterior distribution] & imp>variable importance] & error>mean squarred error];
  class posterior movie;
  class imp movie;
  class error movie;
  posterior & imp & error --> theend([END]);
```
