# CHANGELOG (Release Notes)

## 20358

* Add `upgrade_headers`, a table of header keys and values to be added to the upgrade request.
* Incorporate ping timer from RB
* Fix string sub calls for fragments (one from RB; there was another).
* Defensive socket closes.
* Attempt log message if message handler throws an error.
