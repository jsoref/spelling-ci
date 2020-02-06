# spelling-ci
integration testing support templates

## test-spelling-unknown-words based on fchurn

[fchurn](https://github.com/jsoref/spelling/blob/master/fchurn) is the tool that I personally use to scan repositories for misspellings. It's designed so that I can correct misspellings in a repository, and then come back in a year and only look at new misspellings. Essentially, that's the design that is deployed by `test-spelling-unknown-words`.

### Required Configuration Variables


| Variable | Description |
| ------------- | ------------- |
| bucket | a `gsutil` compatible url for which the tool has read access to a couple of files |
| project      | a folder within `bucket`. This allows you to share common items across projects. |
| spellchecker | The directory where the spell checker lives -- this is where `exclude` lives |
| temp | The directory where the spell checker puts its files |
| wordlist | the path for your base dictionary (`english.words.gz` in the Input Files section). |

### Input files

| Path | Description |
| ------------- | ------------- |
| `gs://example-bucket/english.words.gz` | *gzipped* word list. Note: It doesn't have to actually be English, although your results may vary. |
| `gs://example-bucket/project/` | Project specific storage. |
| `gs://example-bucket/project/excludes` | Perl regular expressions for paths in the repository to ignore. |
| `gs://example-bucket/project/whitelist` | Sorted list of additional tokens which the repository owner has temporarily deemed acceptable. |

### Other files

| Name | Description |
| ------------- | ------------- |
| `english.words` | the dictionary |
| `excludes` | file with perl regular expressions (one per line) to exclude files from scanning |
| `spelling-unknown-word-splitter` | `w` - the word splitter engine itself |
| `unknown.words` | words that were not in the dictionary |
| `whitelist.words` | project specific acceptable words |

### Output

#### New misspellings found
```
New misspellings found, please review:
deps
fchurn
params

If you are ok with the output of this run, you will need to
gsutil cp gs://... ...
patch ... <<EOF
@@ -98,2 +98,3 @@
+fchurn
 fd
@@ -382,2 +384,3 @@
 pageshadow
+params
 parsererror
@@ -541,3 +544,2 @@
 Uf
-ulimit
 unapply
EOF
gsutil cp ... gs://...
```
`script returned exit code 1`

#### Fewer misspellings

```
Clean up from previous run
Run w
Review results
There are now fewer misspellings than before.
.../whitelist.words could be updated:

gsutil cp gs://... ...
@@ -541,3 +544,2 @@
 Uf
-ulimit
 unapply
EOF
gsutil cp ... gs://...
```
`script returned exit code 1`

### Scripts

* [test-spelling-unknown-words](test-spelling-unknown-words) - This script packages together a couple of scripts from my [spelling](https://github.com/jsoref/spelling/tree/04648bdc63723e5cdf5cbeaff2225a462807abc8) repository:

  * [f](https://github.com/jsoref/spelling/blob/04648bdc63723e5cdf5cbeaff2225a462807abc8/f) - a script which looks for files / excludes files and then runs `w` on them.

      * It runs `w` twice because otherwise xargs and perl get overloaded trying to process the input

  * [w](https://raw.githubusercontent.com/jsoref/spelling/master/w) - is mostly a custom word splitter with enough intelligence to ignore words from a wordlist.

      * On its own, it tries to use `/usr/share/dict/words` which is sometimes a useful set, and ofttimes a poor set.
         * `test-spelling-unknown-words.sh` rewrites `w` to use the dictionary you provide.
      * I tend to use the Fedora project's `words` package to get my wordlist. It's somewhere near [https://rpmfind.net/linux/fedora/linux/development/rawhide/Everything/aarch64/os/Packages/w/](https://rpmfind.net/linux/fedora/linux/development/rawhide/Everything/aarch64/os/Packages/w/)
         * I used to retrieve it automatically, but they have changed the compression scheme to `lz` which isn't particularly well supported by old systems, so at this point, I encourage everyone to just locally stash their dictionary.
  * [dn](https://github.com/jsoref/spelling/blob/e5c043f3c429f6497853d5a5bbceec1f477f7e1b/dn) - this script filters diffs to look at only additions (since we're looking for **new** misspellings).

* [exclude](exclude) - this replaces the filtering which is classically managed by `f`. It now reads a file `excludes` and uses those lines to exclude files from being parsed.
  * The primary thing you're trying to exclude are binaries. These could be pictures, movies, executables, or archives. If you deal w/ cryptography or build systems, then you may want to exclude files that are full of hashes or signatures.

### Dependencies

* `bash` (I may at some point replace `bash` with `perl`) -- I may also at some point rewrite this system in JavaScript or Python. My original implementation of this was pure JavaScript (hosted by xpcshell...).
* `cat`
* `curl` -- to retrieve `w`
* `git` (for `git ls-files`) or `hg` -- to list files to check
* `gsutil` -- For retrieving settings (most of the files defined in [Other Files](README.md#Other Files)
* `gunzip` (this is just to save storage+transfer cost from the bucket -- you can of course trade off for a bigger file)
* `perl`
* [https://raw.githubusercontent.com/jsoref/spelling/master/w](https://raw.githubusercontent.com/jsoref/spelling/master/w) -- as a reminder, if you use this repository as is, you are running code directly from my repository. If that's a problem, you will want to make a local copy of `w` and commit it into your repository.
* `xargs`

### Jenkins integration example

```Jenkinsfile
stage("Spelling") {
    withCredentials([file(credentialsId: "spelling-bucket-credentials", variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
        sh label: "Activate Service Account", script: "gcloud auth activate-service-account $SERVICE_ACCOUNT --key-file $GOOGLE_APPLICATION_CREDENTIALS"
    }
    steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
            sh 'bucket=gs://your-bucket-url project=your-project junit=.ci-temp/spelling.xml spelling-ci/test-spelling-unknown-words'
        }
    }
    post {
        always {
            junit allowEmptyResults: true, testResults: '.ci-temp/spelling.xml'
        }
    }
}
```
