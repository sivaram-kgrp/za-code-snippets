# Zoho Analytics JSON/CSV File Upload

## Upload JSON/CSV to Zoho Analytics

[Singular Zoho File Upload Utility method](https://github.com/shopelect-remote-git/finkraft-apis/blob/master/src/utils/zohoFileUploader.ts)

### [Zoho Analytics API Docs - Import Data](https://www.zoho.com/analytics/api/#import-data)

```javascript
import fetch from 'node-fetch';
import FormData from 'form-data';

const zohoAuthToken = process.env.ZOHO_AUTH_TOKEN;
const zohoApiBaseUrl = process.env.ZOHO_API_BASE_URL;
const zohoAccountEmailAddress = process.env.ZOHO_ACCOUNT_EMAIL_ADDRESS;

export enum ImportType {
  APPEND = 'APPEND',
  INSERT = 'INSERT',
  UPDATEADD = 'UPDATEADD',
}

export enum FileType {
  JSON = 'JSON',
  CSV = 'CSV',
}

export interface ZohoFileUploaderProps {
  buffer: Buffer;
  fileName: string;
  workspaceName: string;
  tableName: string;
  createTable?: boolean;
  importType: ImportType;
  matchingColumns?: string | null;
  fileType: FileType;
  dateFormat?: string | null;
}

/**
 * Upload file to Zoho Analytics table.
 */
const zohoFileUploader = async ({
  buffer,
  fileName,
  workspaceName,
  tableName,
  createTable = false,
  importType = ImportType.APPEND,
  matchingColumns = null,
  fileType = FileType.JSON,
  dateFormat = null,
}: ZohoFileUploaderProps) => {
  const formData = new FormData();
  formData.append('ZOHO_FILE', buffer, { filename: fileName });

  const url = `${zohoApiBaseUrl}/${zohoAccountEmailAddress}/${workspaceName}/${tableName}`;

  const queryParams = {
    ZOHO_ACTION: 'IMPORT',
    ZOHO_IMPORT_FILETYPE: fileType,
    ZOHO_IMPORT_TYPE: importType,
    ZOHO_AUTO_IDENTIFY: 'TRUE',
    ZOHO_ON_IMPORT_ERROR: 'SETCOLUMNEMPTY',
    ZOHO_CREATE_TABLE: createTable,
    ZOHO_OUTPUT_FORMAT: 'JSON',
    ZOHO_ERROR_FORMAT: 'JSON',
    ZOHO_API_VERSION: '1.0',
    authtoken: zohoAuthToken,
    ...(dateFormat && { ZOHO_DATE_FORMAT: dateFormat }),
    ...(matchingColumns && { ZOHO_MATCHING_COLUMNS: matchingColumns }),
  };

  const urlWithQueryParams = `${url}?${Object.entries(queryParams)
    .map((AoA) => {
      return `${AoA[0]}=${AoA[1]}`;
    })
    .join('&')}`;

  try {
    const data = await fetch(urlWithQueryParams, {
      method: 'POST',
      body: formData,
    }).then((res) => res.json());
    return data.response;
  } catch (error) {
    console.error(error);
    throw error;
  }
};

export default zohoFileUploader;
```

---

# Fetching Data from Zoho Analytics

## Fetch Data from Zoho Analytics

[Export Data Async Utility method](https://github.com/shopelect-remote-git/vendor-middleware/blob/c70bdd4c2f0fffd5743be07296c11d59855e0bf1/utils/zoho-operations/index.js#L88-L113)

### [Zoho Analytics API Docs - Export Data](https://www.zoho.com/analytics/api/#export-data)

```javascript
/**
 * @function
 * Export data to zoho.
 * @param {object} params - Payload for import data.
 * @param {string} params.table - Table name to import to.
 * @param {string} params.workspace - Workspace name to import to.
 * @param {string} [params.columns='*'] - Columns to exports.
 * @param {string} [params.conditionClause=''] - Where clause for sql.
 * @param {boolean} [params.enp=true] - Columns to exports.
 * @returns {Promise} Promise object represents the response for sqs.
 */
const exportDataAsync = async (params) => {
	const {
		table,
		workspace,
		columns = '*',
		conditionClause = '',
		enp = true,
	} = params;
	const func = enp ? fetchRowsENP : fetchRows;
	return new Promise((resolve, reject) => {
		func(columns, table, workspace, conditionClause, (body, data) => {
			if (body && body.rows) {
				resolve(body);
			} else if (_.isString(body) && body === 'JSONERROR') {
				resolve(data);
			} else {
				reject();
			}
		});
	});
};

// Sample Method Call
exportDataAsync({
	table: 'CG data',
	workspace,
	columns:
		'EmployeeCode, EmployeeName, RecipientGSTIN, SupplierGSTIN, CheckIn, CheckOut, HotelName, VendorAddress, TaxableValue',
})
	.then((resp) => {
		if (!resp.rows.length) {
			return res.status(200).json(successRespObj([]));
		}

		const data = formatZohoData(resp.column_order, resp.rows);

		return res.status(200).json(successRespObj(data));
	})
	.catch((err) => {
		res.status(500).json(failureRespObj(err));
	});
```

### Fetch Rows Utilty method

```javascript
const fetchRows = (
	columnList,
	table,
	workspaceName,
	conditionalClause,
	callback
) => {
	let sql = `select ${columnList} from [${table.replace(/['"]+/g, '')}]`;
	if (conditionalClause) {
		sql = `${sql} ${conditionalClause}`;
	}
	const params = {
		ZOHO_ACTION: 'EXPORT',
		ZOHO_SQLQUERY: sql,
		ZOHO_OUTPUT_FORMAT: 'JSON',
		ZOHO_ERROR_FORMAT: 'JSON',
		ZOHO_API_VERSION: '1.0',
	};

	zohoRestCall(
		`${ZOHO_EMAIL}/${workspaceName}`,
		params,
		(err, response, body) => {
			if (!err) {
				if (response.statusCode === 200) {
					try {
						const json = JSON.parse(body);
						if (json.response.result) {
							callback(json.response.result);
						}
					} catch (_err) {
						callback('JSONERROR', body);
					}
				} else {
					console.log(`response isnt 200 ${workspaceName}: ${body}`);
					callback();
				}
			} else {
				console.log(`Error: ${workspaceName}: ${JSON.stringify(err)}`);
				callback();
			}
		}
	);
};
```

### Fetch rows ENP utility method (Rate limited to 1 invocation per 3 seconds)

```javascript
const fetchRowsENP = limit(
	(columnList, table, workspaceName, conditionalClause, callback) => {
		let sql = `select ${columnList} from [${table.replace(/['"]+/g, '')}]`;
		if (conditionalClause) {
			sql = `${sql} ${conditionalClause}`;
		}
		console.log(sql);
		const params = {
			form: {
				ZOHO_OUTPUT_FORMAT: 'JSON',
				ZOHO_ERROR_FORMAT: 'JSON',
				ZOHO_API_VERSION: '1.0',
				authtoken: ZOHO_ENP_AUTH_TOKEN,
				ZOHO_ACTION: 'EXPORT',
				ZOHO_SQLQUERY: sql,
			},
		};

		request.post(
			`${ZOHO_ENP_BASE_URL}/${ZOHO_ENP_EMAIL}/${workspaceName}?`,
			params,
			(error, response, body) => {
				if (!error) {
					if (response.statusCode === 200) {
						try {
							const json = JSON.parse(body);
							if (json.response.result) {
								callback(json.response.result);
							}
						} catch (_err) {
							callback('JSONERROR', body);
						}
					} else {
						console.log(`response isnt 200 ${workspaceName}: ${body}`);
						callback();
					}
				} else {
					console.log(`Error: ${workspaceName}: ${JSON.stringify(error)}`);
					callback();
				}
			}
		);
	}
)
	.to(1)
	.per(3000);
```

### Generic REST API Call method for ZA interface.

```javascript
const zohoRestCall = async (url, optionalParams, callback) => {
	const params = {
		form: {
			ZOHO_OUTPUT_FORMAT: 'JSON',
			ZOHO_ERROR_FORMAT: 'JSON',
			ZOHO_API_VERSION: '1.0',
			authtoken: ZOHO_AUTH_TOKEN,
		},
	};
	Object.keys(optionalParams).forEach((key) => {
		params.form[key] = optionalParams[key];
	});
	request.post(`${ZOHO_BASE_URL}/${url}?`, params, (error, response, body) => {
		callback(error, response, body);
	});
};
```

---

## [Zoho Analytics API Usage Limits](https://www.zoho.com/analytics/api/#api-usage-limits-amp-pricing)
