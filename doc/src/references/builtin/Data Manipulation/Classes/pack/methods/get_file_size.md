# get_file_size
Returns the size of a particula file in the pack.

`uint pack::get_file_size(string filename);`

## Arguments:
* string filename: the name of the file to query the size of.

## Returns:
uint: the size of the file (in bytes), or 0 if no file with that name was found.