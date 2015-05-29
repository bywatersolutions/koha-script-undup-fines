# NAME

undup\_fines.pl

# SYNOPSIS

./undup\_fines.pl \[-c\] \[-d\[=0\]\] \[-n\[=0\]\] \[-r\[=REPORT FILE NAME\]\] 
./undup\_fines.pl \[-h\]

# OPTIONS

- **-c|--commit**

    undup\_fines.pl will not make any changes to the fines table unless the
    '-c' flag is specified.

- **-d|--drop**

    Drop temp table after program has completed. This is the default option,
    use '-d=0' if you do not wish to drop the table.

- **-n|--new\_table**

    Create temp table when the program starts. This this is the default
    option, use '-n=0' if you do not wish to create the table for faster
    startup. The table **must** exist for the program to run. This can be
    accomplished by running the program with the the '-d=0' option.

- **-r|--report\_file**

    The program will create a CSV file containing a list of the actions
    taken. The file name defaults to '~/undup\_fines\_report.csv' if not
    specified.

- **-h|--help**

    Print this help message and exit.

# DESCRIPTION

## The problem being fixed

The duplicates were caused by a bug in C4::Overdues::UpdateFine()
where fines to be updated were looked up by description. 

The descriptions for fines include the due date/time of the item.
All items are due at one minute before midnight.  UpdateFine() did
not take into account changes in time format, as specified using the
'TimeFormat' system preference. The 'TimeFormat' system preference may
be set to '24H' or '12H'. Time due would be set to '23:59' or '11:59 PM'
respectively. Changing TimeFormat did not re-set the time formats within
the descriptions, so UpdateFine created duplicate fines for all fines
updated where a fine description had the old time format.

This function would update fines with the following account types:

    F   | Fine
    FU  | Fine Update
    O   | Overdue
    M   | Miscelaneous

For our purposes, fines with a time format which does not match
the current TimeFormat will be considered 'BAD' and fines which
match will be considered 'GOOD'.

For our purposes, we will have to consider the following variables:

    * Are there duplicate fines?
    * Is the fine GOOD or BAD?
    * Has the fine been paid?

If there are no duplciate fines:
    Leave GOOD fines alone
    Change the date format on BAD fine to make it GOOD.

If there are duplicate fines:
    For each BAD fine:
        Delete the BAD fine.
        If the fine has been paid: 
           reduce the amount\_outstanding on the corresponding GOOD fine.
        If the amount\_outstanding on the GOOD fine is negative, set it to 0, and send a warning to the logs.

Fines are considered duplicates if the following conditions are true:

    * The descriptions differ by only the time format at the end.
    * borrowernumber matches
    * itemnumber matches

In the case of serials, it is possible that description and borrowernumber
will match, but itemnumber is NULL. In these cases, the records will be
sent to the report file, but no changes will be made automatically. These
will require manual modification by staff.

## The procedure of fixing the problem.

Read the 'TimeFormat' system preference

Create a temporary fines table 'temp\_duplicate\_fines' containing 
any fines where the account types are in "F", "FU", "O", "M".

The temporary table will have the following information:

    accountlines_id        | Link back to accountlines
    original_description   | Including the offending time
    my_description         | Without the time part.
    timestamp              | Timestamp from accountlines
    date                   | Date from accountlines
    amount_paid            | amount - amountoutstanding
    correct_timeformat     | 0 for 'BAD', 1 for 'GOOD'.

There should only be at most two rows for each description -- 
timeformat will be 0 or 1.

    For each fine matching the criteria above
        If borrowernumber, description or itemnumber is missing:
            Send a  warning to the logs if
                The description is BAD or
                The description is GOOD, but might be paired with a BAD description.
        If borrowernumber, description or itemnumber are all present
            Add row to the temporary table.
    For each pair of rows in the temporary table having the same borrowernumber, my_description and itemnumber
        For each pair, there must be a newer and an older fine. If the fines have the same date, we don't know which to keep; these must be resolved by hand.
        Run a query to find which data to keep:
            Amount
            Amount outstanding
            Description (this will always be the 'Good' description)
            Accounttype
            Date (Always the earlier of the pair of dates)
            Lastincrement
        If the amount paid is more than the amount, set Amount_outstanding to 0, and log a warning.
    For singletons (Fines without a duplicate)
        If the singleton has a 'Bad' description, update it.
    Check for multiple fines having the same borrowernumber, my_description and itemnumber. There shouldn't be any.
    We loop throught the list of data to keep
        Update the record of data to keep
        Delete the other record

# WARNINGS

- "Accountlines description $my\_description matches record with undefined fields. Please inspect."

    This program only makes changes to fines that have the incorrect date
    format. If one of these fines is missing borrowernumber, description
    or itemnumber, this doesn't matter unless the item matches a fine that
    does have an incorrect date format. This warning is an indication that
    there is a record with correct date format that is missing one of these
    fields. In this case the duplicates must be cleared manually.

- "Accountlines record is missing \[borrowernumber, description, itemnumber\]. Please inspect."

    A fine with incorrect time format is missing borrowernumber, description
    or itemnumber fields. This must be fixed manually.

- "Amount paid is greater than amount outstanding"

    This program does not automatically create credits if the amount paid
    is more than the fines incurred. The duplicate fines will be fixed,
    but it is up to the library to decide how to credit over-payments.

- "Duplicates with the same date -- these will need to be handled manually"

    The program must be able to determine which came first: the BAD fine
    or the GOOD. If both fines records were created on the same day,
    the program cannot determine this, and the duplicate fines must be
    cleared manually.

- "There are N records with the following borrowernumber, itemnumber and description" 

    You shuold not see this message -- there should only be pairs of fines,
    one GOOD and one BAD. If you do see this message, the program has failed
    a sanity check, and the fines must be cleared by hand.

# AUTHOR

Barton Chittenden <barton@bywatersolutions.com>

# LICENSE

This file has the same license as Koha.

Koha is free software; you can redistribute it and/or modify it under
the terms of the GNU General Public License as published by the Free
Software Foundation; either version 2 of the License, or (at your option)
any later version.

You should have received a copy of the GNU General Public License
along with Koha; if not, write to the Free Software Foundation, Inc.,
51 Franklin Street, Fifth Floor, Boston, MA 02110-1301 USA.

# DISCLAIMER OF WARRANTY

Koha is distributed in the hope that it will be useful, but WITHOUT
ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or
FITNESS FOR A PARTICULAR PURPOSE.  See the GNU General Public License
for more details.
