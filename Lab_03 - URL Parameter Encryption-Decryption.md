# How to Encrypt and Decrypt URL Parameter(s)
## Step 1: Prepare a class containing methods for Encryption and Decryption 
- Here we have created a `static class UrlEncryptor` with two static methods.
- **Encrypt()** - accepts a string and returns encrypted key
- **Decrypt()** - accepts an encrypted key and returns the orginal string.
- Here we required an Encryption Key which is 16 character alphanumeric string.
```csharp
public static class UrlEncryptor
    {
        private static readonly string EncryptionKey = "pjsGLNYrMqU6wny4"; // Change this key

        public static string Encrypt(string text)
        {
            using (var aesAlg = Aes.Create())
            {
                aesAlg.Key = Encoding.UTF8.GetBytes(EncryptionKey);
                aesAlg.IV = new byte[16]; 

                var encryptor = aesAlg.CreateEncryptor(aesAlg.Key, aesAlg.IV);

                using (var msEncrypt = new MemoryStream())
                {
                    using (var csEncrypt = new CryptoStream(msEncrypt, encryptor, CryptoStreamMode.Write))
                    using (var swEncrypt = new StreamWriter(csEncrypt))
                    {
                        swEncrypt.Write(text);
                    }

                    return Convert.ToBase64String(msEncrypt.ToArray());
                }
            }
        }

        public static string Decrypt(string encryptedText)
        {
            using (var aesAlg = Aes.Create())
            {
                aesAlg.Key = Encoding.UTF8.GetBytes(EncryptionKey);
                aesAlg.IV = new byte[16];

                var decryptor = aesAlg.CreateDecryptor(aesAlg.Key, aesAlg.IV);

                using (var msDecrypt = new MemoryStream(Convert.FromBase64String(encryptedText)))
                using (var csDecrypt = new CryptoStream(msDecrypt, decryptor, CryptoStreamMode.Read))
                using (var srDecrypt = new StreamReader(csDecrypt))
                {
                    return srDecrypt.ReadToEnd();
                }
            }
        }
    }
```
## Step 2: Encrypt URL as and when required
Here we have encrypted URL during Edit operation as following
> Note: Only Logic mentioned over here
```csharp
if (Model.Rows.Count > 0)
{
    foreach (DataRow dr in Model.Rows)
    {
        <tr>
            <td>@dr["ProductName"]</td>
            <td>@dr["ProductPrice"]</td>
            <td>@dr["ProductCode"]</td>
            <td>@dr["Description"]</td>
            <td>
                <a asp-controller="Home" asp-action="Delete" class="btn btn-danger btn-xs" asp-route-ProductID="@UrlEncryptor.Encrypt(dr["ProductID"].ToString())">Delete</a>
                <a id="btnEdit" class="btn btn-info btn-xs" asp-controller="Home" asp-action="Add" asp-route-ProductID="@UrlEncryptor.Encrypt(dr["ProductID"].ToString())">Edit</a>
            </td>
        </tr>
    }
}
```
## Step 3: Decrypt URL as and when required
Here we have decrypted the string using Decrypt() method as following
> Note: Only Logic mentioned over here
```csharp
public IActionResult Delete(string ProductID)
{
    try
    {
        int decryptedProductID = Convert.ToInt32(UrlEncryptor.Decrypt(ProductID.ToString()));
        SqlConnection objConn = new SqlConnection(this.Configuration.GetConnectionString("CoffeeShop"));
        objConn.Open();
        SqlCommand objCmd = objConn.CreateCommand();
        objCmd.CommandType = CommandType.StoredProcedure;
        objCmd.CommandText = "PR_Product_DeleteByPK";
        objCmd.Parameters.Add("@ProductID", SqlDbType.Int).Value = decryptedProductID;

        if (objCmd.ExecuteNonQuery() > 0)
        {
            TempData["Message"] = "Record Deleted Sucessfully";
        }
    }
    catch(Exception ex)
    {
        TempData["Message"] = ex.Message.ToString();
    }
    finally
    {
        objConn.Close();
    }

    return RedirectToAction("Index");
}
```
