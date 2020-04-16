# csv2json

**Command-line tool to convert CSV to JSON built in PHP.**

This is an attempt at [Frédéric Bouchery](https://twitter.com/fredbouchery/) exercise: (in french) [https://gist.github.com/f2r/2f1e1fa27186ac670c21d8a0303aabf1](https://gist.github.com/f2r/2f1e1fa27186ac670c21d8a0303aabf1)

*Disclamer: This is very experimental and may contains many pitfalls*

## Requirements

* PHP `7.4+`
* ext-json

## Features and limitations

**Features**
* Portable (command and unit-tests): tested on Linux and Windows (Not tested on OSX, but it should works :D)
* No tooling, no vendor, only need PHP `7.4+` and `json` extension
* One file, ~350 LOC
* Read from stdin or file, can be used as a linux pipe `cat file.csv | csv2json --fields ... - | ...`
* Output (json) on STDOUT
* Errors on STDERR, different non-zero exit codes based on the error
* Automatic detection of the CSV separator (Im not totally convinced by the solution, but it should works with most of CSVs :see_no_evil:)
* No memory overhead (peak around 25MB), no matter the csv size, thanks to generators. **Without `--aggregate` option**
* Can recover ~~pretty well~~ on corrupted lines or format (`--ignore` option)
* Skip empty lines
* Try to play nice with int/float formatting (ex: `1 000 250,20` -> `1000250.20`), no locale detection whatsoever, very naive

**Limitations**
* The automatic delimiter detection is fragile, and there no way to specify the delimiter (cli option or use desc file for example, but yolo)
* Default enclosure `"` and escape char `\`, not auto detected and no way to specify them
* No BOM handling
* Very basic, no check of all the CSV pitfalls
* `--aggregate` needs to keep in memory all the parsed lines (to group by a field): memory explosion, depending of the csv size (see [benchmark](#benchmark))
* Very basic date/time formatting (just a validation of a format)
* The obstinacy to keep only one file forces to use some little tricks. One obvious solution is to build a PHAR, but with that comes tooling or vanilla glue code to build the PHAR: more problems to manage for no or little value (if the project keep this size)
* Many others things for sure

## Usage

```
Command-line tool to convert CSV to JSON

    Usage: csv2json [options] <csv_file>
    
    Examples:
        csv2json myfile.csv > result.json
        echo "id;name\n1;bob" | csv2json -
    
    Available options:
      -h                              Show this help text
      --fields "<field1,field2,...>"  List of field (based on headers) to include in the json. All by default
      --aggregate <field>             Aggregate lines on the selected field
      --desc <file>                   Description file of fields types. INI format: field1=?bool (? before the type to convert empty cell in null)
                                      Supported formats: string, float, int, bool, date, time, datetime
      --pretty                        Print human readable json
      --ignore                        Ignore errors of inconsistent line size or format, useful for trashy CSV
```

## Benchmark

Some benchmarks (I am no expert at all in this field)

Test environment:
* PHP 7.4.4 (cli) (built: Mar 20 2020 13:47:17)
* Hardware
    * Intel(R) Core(TM) i7-7700HQ CPU @ 2.80GHz 8 cores (1 core is used by the command anyway)
    * Disk: 250GB SSD, RAM: 16GB
    * OS: Linux Mint 18.3 Sylvia
* CSV file: `180Mo`, `~390K` lignes, `25` columns (string, int, date, bool, etc)

**[gnu time](https://www.gnu.org/software/time/) simple bench**

| command | time | memory peak | json produced |
| -------- | ---- | ----------- | ------------- |
| `csv2json file.csv` | ~8.2s | 25MB | 294Mo |
| `csv2json --fields id,type,email,date, file.csv` | ~8.5s | 25MB | 43Mo |
| `csv2json --desc desc.ini file.csv` 5 fields described (int, float, bool) | ~9.0s | 25MB | 296Mo (because of `null` instead of empty string?) |
| `csv2json --aggregate type file.csv` | ~8.6s | 1.288GB | 286Mo |
| `csv2json --desc desc.ini --aggregate type file.csv` | ~9.0s | 1.25GB | 288Mo |

**Blackfire.io profiling**

| command | time | memory peak | link to blackfire profile |
| -------- | ---- | ----------- | ------------------------- |
| csv2json file.csv | 9.92s | 148kB | [profile](https://blackfire.io/profiles/66af0af3-aca3-4173-b1ca-01fab9706d51/graph) |
| csv2json --desc desc.ini file.csv | 11.3s | 148kB | [profile](https://blackfire.io/profiles/3ee9f992-1a91-4497-bfd9-9e1ac0d29968/graph) |
| csv2json --desc desc.ini --aggregate type file.csv | 14.3s | 1210MB | [profile](https://blackfire.io/profiles/b7beb70f-307e-4c15-bf56-353eadc59331/graph) |

## Unit tests

*Disclaimer: The test suite is far from complete, time is missing, but the spirit is here :)*

*Second disclaimer: this "unit" test suite also contains end-to-end test (real execution of the command and test output/err, and code)* 

Here again, this is a custom test suite in plain "vanilla" PHP.
To run the test suite, use the command below:

    ./unit-test
    # or
    php unit-test

Output example:
```
✗ Test 'formatFloat' with dataset #1 failed in unit-test line 13:
    Exception raised: Actual value 1000.85 does not equal expected value 1000.84

✗ Test 'formatFloat bad format' with dataset #0 failed in unit-test line 20:
    Exception raised: Exception message "Value is not numeric" doesn't matches "/not numeric yolo/"

Tests executed: 3
✖ Test suite failed: 1 SUCCESS, 2 FAILED
```

## Credits

Thanks to Thierry Geindre ([@t_geindre](https://twitter.com/t_geindre)) for his advice, help and review :)