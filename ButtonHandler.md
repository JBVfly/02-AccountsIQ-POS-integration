
Handle the various buttons on the main CSV import screen

``` C#

using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Data;
using System.Drawing;
using System.Linq;
using System.Text;
using System.Windows.Forms;
using ImpPrototype.ServiceReference1 ; 
//using Tutorial_A.ServiceReference1;  // ServiceReference1; 
using System.IO ;  // Added 16 Mar '17
using System.ServiceModel ; // Added 16 Mar '17
using System.Diagnostics;
using System.Data.OleDb;    // Added 9 Mar '17


namespace ImpPrototype
{
    public partial class ImportForm2 : Form
    {
        public String sLoc = "???" ;
        public FileCSV oLocSel ;

        private int m_SelIndex ;

        //Integration_1_1SoapClient WebServ = new Integration_1_1SoapClient();
        private Integration_1_1SoapClient m_WebServ = new Integration_1_1SoapClient();

        // Keep track of amounts with assigned acct and the balance - - -
        private float m_Balance;
        private int m_AcctAssignCnt; 

        // Moving data into this class with new 3-pack method - - - - - - - -
        public bool Post2Test { get; set; } // <-- 14 May '17 Post to test company 
        public struct SetStruc
        {
            public string Desc ;
            public float fAmt  ;
            public string DrAcct  ;
            public string CrAcct  ;
            public string RptLine ;
            public bool InMDB ; 
        }        
        public int SetStat { get; set; }   // 1 or -1  (3-pack or missing a file or two)
        public String[] aSetDesc = new string[150] ;
        public float[]  aSetAmt  = new float[150]  ;
        public int SetSize { get; set; }   
        public String sSetDate { get; set; }
        public DateTime oSetDate { get; set; }
        public String sSetLocation { get; set; }
        public SetStruc[] aSetVals = new SetStruc[150];

        private string sSetLocCode ;    // <-- 7 May '17  Location code by itself 
        private string sCurrDateTime ;  // <-- 7 May '17  For testing 

        private Dictionary<string, string> dictAccts  = new Dictionary<string, string>() ;
        private SortedList<string, string> SListAccts = new SortedList<string, string>() ;

        // List of transactions to post - - - - - - -
        public struct TranStruc
        {
            public string Acct ;
            public decimal Amt ;
            public string Desc ; 
        }
        List<TranStruc> listTran = new List<TranStruc>() ;

        // 7 May '17 - A dictionary for accts that have already been mapped 
        /*public struct MapStruc
        {
            public string DrSide;
            public string CrSide;
            public string Desc;
        }
        private Dictionary<string, MapStruc> dictMapAccts = new Dictionary<string, MapStruc>() ; */
        // MDB 
        private OleDbConnection m_conn = new OleDbConnection() ;
        private String m_connStr;
        private string m_sSusAcct ;

        private decimal m_SusAmt ;  // <-- Retain suspence amount to write to API post log 

        public struct KeyStruc
        {
            public string CompID   ;
            public string keyPart  ;
            public string keyUser  ;
            public string CompName ;
        }
        private KeyStruc m_APIkey ;

        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = 
        public ImportForm2()
        {
            InitializeComponent();
            //textBoxStat.AppendText("After Initialize");
            m_connStr = @"Provider=Microsoft.ACE.OLEDB.12.0;" ;
            m_connStr += "Data Source=AppData.mdb;" ;
            m_conn.ConnectionString = m_connStr ;
        }
        
        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = 
        private void ImportForm2_Load(object sender, EventArgs e)
        {
            tbLoc.Text  = sSetLocation ; 
            tbDate.Text = sSetDate     ;

            // 7 May '2017 - Seperate out the code in the beginning on the location info 
            int iPos = sSetLocation.IndexOf(" ") ;
            sSetLocCode = sSetLocation.Substring(0, iPos) ; 

            listViewAccts.View = View.Details  ;
            listViewAccts.GridLines = true     ;
            listViewAccts.FullRowSelect = true ;

            //textBoxStat.AppendText("\r\nColumns");
            listViewAccts.Columns.Add("Description", 550);
            listViewAccts.Columns.Add("Amount",  85, HorizontalAlignment.Right)  ;
            listViewAccts.Columns.Add("Debit",  100, HorizontalAlignment.Center) ;
            listViewAccts.Columns.Add("Credit", 100, HorizontalAlignment.Center) ;

            if (Post2Test)
                textBoxStat.AppendText("Post to test company") ;
            else
                textBoxStat.AppendText("Post to LIVE company") ;

            cbDebit.DropDownStyle  = ComboBoxStyle.DropDownList ;    // <-- Set style of comboboxes 
            cbCredit.DropDownStyle = ComboBoxStyle.DropDownList ;
            btnSave.Enabled = false ;

            //m_sCompID = "SNR3" ;
            //m_sCompID = "SNR6";  // <-- Hammond property (Test Import Company)
            //m_keyPart = "ZGfaC2nJm1awEx+i3Z+Fzf.....";
            //m_keyUser = "8KYbmseEdiDxCJU8+9VxsbgRjQFPkYDcJ9kAjMocY...." ;
            // Hammond property (Test Import Company)
            //m_keyUser = "8KYbmseEdiDxCJU8+9VxsbgRjQFP....";

            GetAPIkeys() ;
            if (AIQ_ChartOfAccts_etc())
            {
                GetAcctMap() ; // <-- 7 May '17 Build acct map 
                PopulateCSVListView() ;
            }

         }  // private void ImportForm2_Load(object sender, EventArgs e)


        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = 
        private void GetAPIkeys()
        {
            m_APIkey.CompID  = "SNR6956" ;
            m_APIkey.keyPart = "ZGfaC2nJm1awEx+i3Z+FzfIFr3nho9Rw+KwW4Ce..." ;
            m_APIkey.keyUser = "8KYbmseEdiDxCJU8+9VxsbgRjQFPkYDcJ9kAjMocYr4WqDFm...." ;
            m_APIkey.CompName = "..." ;
        }

        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = 
        private void GetAcctMap()
        {
            /*
             *  Read Desc/Accts that have already been matched 
             *        and add to Set array
             * 
             */
            textBoxStat.AppendText("\r\nBuildAcctMap()");

            OleDbCommand select = new OleDbCommand();

            if (SetStat > 0)  // <-- Make sure we have a complete 3-pack 
            {
                try
                {
                    m_conn.Open() ;
                    //textBoxStat.AppendText("\r\n Open .MDB must have worked, yea! ");

                    select.Connection = m_conn;
                    select.CommandText = string.Format("Select FullDesc, DrSide, CrSide From AcctMap Where location = '{0}' ",
                                                        sSetLocCode);
                    //textBoxStat.AppendText(string.Format("\r\n >{0}< ", select.CommandText));
                    OleDbDataReader reader = select.ExecuteReader();

                    //if (reader.HasRows)
                    //    textBoxStat.AppendText("\r\nAcctMap has rows") ;
                    //else
                    //    textBoxStat.AppendText("\r\nNo rows in AcctMap") ;

                    string sDesc ;
                    //bool bSuspence ; 
                    while (reader.Read())
                    {
                        sDesc = reader.GetString(0) ; 
                        //textBoxStat.AppendText(string.Format("\r\nfd:{0}", sDesc)) ;
                        //bSuspence = false ;
                        for (int iLnSub = 0; iLnSub < SetSize; iLnSub++)
                        {
                            if (aSetVals[iLnSub].Desc.CompareTo(sDesc) == 0)
                            {
                                //textBoxStat.AppendText(string.Format(" Match at {0}", iLnSub.ToString())) ;
                                //aSetVals[iLnSub].DrAcct = reader.GetString(1) ;
                                aSetVals[iLnSub].DrAcct = VerifyAcct(reader.GetString(1)) ;
                                //aSetVals[iLnSub].CrAcct = reader.GetString(2);
                                aSetVals[iLnSub].CrAcct = VerifyAcct(reader.GetString(2)) ;
                                aSetVals[iLnSub].InMDB = true; // <-- Note acct was previously mapped and in MDB
                                break ; 
                            }
                        } // for (int iLnSub = 0; iLnSub < SetSize; iLnSub++)
                        // If we make it down here then Desc is not in Set Array, so most likely it is Suspence acct
                        if (sDesc.CompareTo("~SUSPENCE") == 0)
                        {
                            //m_bSusAcctRead = true;
                            m_sSusAcct = VerifyAcct(reader.GetString(1)) ;  // <-- The suspence acct is always stored in DrSide
                            //cbSuspence.Items.Clear() ;
                            //cbSuspence.Items.Add(m_sSusAcct) ;
                            //cbSuspence.SelectedIndex = 0 ;
                            cbSuspence.DropDownStyle = ComboBoxStyle.DropDownList ;
                        }
                    } // while (reader.Read())
                }
                catch (Exception ex)
                {
                    textBoxStat.AppendText(string.Format("\r\n Open must have failed ({0})", ex.Message));
                    //MessageBox.Show("Failed to connect to data source");
                    //textBox1.AppendText("FAILED TO CONNECT \r\n") ;
                    //textBox1.AppendText("Ex:" + ex.Message + "\r\n") ;
                    //m_Stat = -2;
                    //OLE_exceptStr = ex.Message;
                }
                finally
                {
                    m_conn.Close();
                }

                // 17 May '17 - Go ahead and get to Suspence acct 
                if (m_sSusAcct.CompareTo(" ") > 0)
                    cbSuspence.SelectedIndex = SListAccts.IndexOfKey(m_sSusAcct);
            } //if (SetStat > 0)
        } // private void BuildAcctMap()

        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = 
        private string VerifyAcct(string inAcct)
        {
            /*
             * Made sure acct stored in map is currently valid (in chart of accts) 
             * 
             */
            string sRet = "" ;

            if (SListAccts.ContainsKey(inAcct))  
                sRet = inAcct ;   // <-- Key is found to send it back 

            return sRet ; 
        }


        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = 
        private void PopulateCSVListView()
        {
            /*
             * Render/populate list box with CSV data
             *    and also build set of transactions where acct numbers are assgined
             */
            //textBoxStat.AppendText("\r\nPopulateCSVData3()") ;
            string[] arr = new string[4];

            tbAIQcomp.Text = m_APIkey.CompName ;  // <-- 17 May '17 Populate company name   

            ListViewItem itm ;

            m_Balance = 0 ;
            m_AcctAssignCnt = 0 ;
            bool bAssign; 

            //textBoxStat.AppendText(string.Format("\r\niSetSize:{0}", SetSize)) ;
            TranStruc JournalLine ; 
            bAssign = false ;
            listTran.Clear() ; 
            listViewAccts.Items.Clear() ;
            //string sDescFormat ;
            for (int iLnSub = 0; iLnSub < SetSize; iLnSub++)
            {
                //sDescFormat = string.Format("{0} #{1}#", aSetVals[iLnSub].Desc, aSetVals[iLnSub].RptLine) ;
                arr[0] = aSetVals[iLnSub].Desc ;    // aSetDesc[iLnSub];
                arr[1] = string.Format("{0:0.00}", aSetVals[iLnSub].fAmt) ; // oLocSel.asAmt[iLnSub];
                arr[2] = aSetVals[iLnSub].DrAcct ;  //  " ";  //  oLocSel.asDebit[iLnSub];
                arr[3] = aSetVals[iLnSub].CrAcct ;  //  " ";  // oLocSel.asCredit[iLnSub];
                itm = new ListViewItem(arr)  ;
                listViewAccts.Items.Add(itm) ;
                if (aSetVals[iLnSub].DrAcct.CompareTo("0000-00-000") > 0)
                {

                    m_Balance += aSetVals[iLnSub].fAmt ;
                    JournalLine.Acct = aSetVals[iLnSub].DrAcct ;
                    JournalLine.Amt  = Convert.ToDecimal(aSetVals[iLnSub].fAmt) ;
                    //JournalLine.Desc = string.Format("{0}-{1}", sSetLocCode, aSetVals[iLnSub].RptLine) ;  // <-- 7 May '17 
                    JournalLine.Desc = aSetVals[iLnSub].RptLine ;  // <-- 7 May '17 
                    listTran.Add(JournalLine);
                    bAssign = true ;
                }
                if (aSetVals[iLnSub].CrAcct.CompareTo("0000-00-000") > 0)
                {
                    m_Balance += aSetVals[iLnSub].fAmt * -1 ;
                    JournalLine.Acct = aSetVals[iLnSub].CrAcct ;
                    JournalLine.Amt  = Convert.ToDecimal(aSetVals[iLnSub].fAmt*-1) ;
                    //JournalLine.Desc = string.Format("{0}-{1}", sSetLocCode, aSetVals[iLnSub].RptLine) ;  // <-- 7 May '17 
                    JournalLine.Desc = aSetVals[iLnSub].RptLine ;  // <-- 7 May '17 
                    listTran.Add(JournalLine);
                    bAssign = true ;
                }
                if (bAssign)
                    m_AcctAssignCnt++ ;  // <-- This acct was assigned a Dr or Cr
            }  //  for (int iLnSub = 0; iLnSub < SetSize; iLnSub++)
            tbBalance.Text = string.Format("{0:0.00}", m_Balance) ;

            /*textBoxStat.AppendText("\r\nPost accts - - - - - - -") ;
            foreach (TranStruc oTran in listTran)
            {
                textBoxStat.AppendText(string.Format("\r\n post {0} {1:0.00} d:{2}<  ",
                                    oTran.Acct, oTran.Amt, oTran.Desc)) ; 
            }*/

            ChkBoxSuspence() ; // <-- See if Suspence combobox needs to be enabled  
            ChkButtonPost()  ;
        }  //private void PopulateCSVData()


        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = 
        private bool ChkBoxSuspence()
        {
            /*
             *  Should be only enabled when we have an out-of-balance condition 
             * 
             */
            bool bRet = false ;

            cboxSuspence.Enabled = false ;
            cbSuspence.Enabled   = false ;
            if (m_Balance != 0)  // SetStat > 0 && m_AcctAssignCnt > 0)
            {
                cboxSuspence.Enabled = true ;
                if (cboxSuspence.Checked) 
                    cbSuspence.Enabled = true ; 
            }

            return bRet ;
        }  // private bool PostButtonStat()


        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = 
        private bool ChkButtonPost()
        {
            /*
             * See if POST button needs to be enabled  
             * 
             */
            bool bRet = false;

            btnPost.Enabled = false ;

            if (m_AcctAssignCnt > 0)
            {
                if (m_Balance == 0)
                    btnPost.Enabled = true ;
                else
                {
                    if (cboxSuspence.Checked && 
                             cbSuspence.SelectedIndex >= 0 &&
                             !cbSuspence.SelectedItem.ToString().StartsWith("0000-00") ) 
                        btnPost.Enabled = true ;
                }

            } // if (m_AcctAssignCnt > 0)

            return bRet;
        }  // private bool PostButtonStat()


        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = 
        private void listViewAccts_DoubleClick(object sender, EventArgs e)
        {
            /*  
             *   Desc/amt has been double-clicked on...prepare to assign Dr/Cr 
             *   
             */

            if (SetStat > 0)  // <-- Only allow edit for SetStat 1 (full data set) 
            {
                m_SelIndex = listViewAccts.SelectedItems[0].Index;
                //textBoxStat.AppendText("DoubleClick \r\n") ;
                //textBoxStat.AppendText("Idx:" + m_SelIndex.ToString());
                tbDesc.Text = listViewAccts.SelectedItems[0].SubItems[0].Text ;  // <-- Display acct description for editing 
                tbAmt.Text  = listViewAccts.SelectedItems[0].SubItems[1].Text ;  // <-- Display amount from CSV

                //int iAcctCnt = dictAccts.Count ;
                //textBoxStat.AppendText("dictAccts:" + iCnt + "\r\n") ;
                //if (dictAccts.Count <= 0)
                if (SListAccts.Count <= 0)
                {
                    // Get chart of accounts if we don't have them already   
                    //AIQ_ChartOfAccts_etc() ;
                    cbDebit.DropDownStyle    = ComboBoxStyle.DropDownList ;
                    cbCredit.DropDownStyle   = ComboBoxStyle.DropDownList ;
                    cbSuspence.DropDownStyle = ComboBoxStyle.DropDownList ; 
                }

                // Set combobox to existing acct if one has already been assigned
                string sAcct ;
                sAcct = aSetVals[m_SelIndex].DrAcct ;
                if (sAcct.CompareTo(" ") > 0)
                    cbDebit.SelectedIndex  = SListAccts.IndexOfKey(sAcct) ;
                sAcct = aSetVals[m_SelIndex].CrAcct ;
                if (sAcct.CompareTo(" ") > 0)
                    cbCredit.SelectedIndex = SListAccts.IndexOfKey(sAcct) ;
                
                cbDebit.Enabled  = true ;
                cbCredit.Enabled = true ;
                btnSave.Enabled  = true ; 
            } // if (SetStat > 0)
        }  // private void listViewAccts_DoubleClick(object sender, EventArgs e)
        

        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = 
        private bool AIQ_ChartOfAccts_etc()
        {
            /*
             *  Get chart of ACCOUNTS and COMPANY NAME for the given key 
             * 
             */
            bool bRet = false ;  
            //textBoxStat.AppendText("\r\nContacting AIQ for chart of accounts...");
            try
            {
                WaitForm fWait = new WaitForm("Contacting AIQ...");
                fWait.Show(); 
                string sAuth = m_WebServ.Login(m_APIkey.CompID, m_APIkey.keyPart, m_APIkey.keyUser);  // m_sCompID, m_keyPart, m_keyUser);
                //sAuth = m_WebServ.Login(m_sCompID, m_keyPart, m_keyUser) ;  // m_sCompID, m_keyPart, m_keyUser);
                if (sAuth == null)
                    textBoxStat.AppendText("   AIQ Login failed ");
                else
                {
                    bRet = true ;
                    //textBoxStat.AppendText("Auth not null \r\n") ;
                    //textBoxStat.AppendText("Auth:" + m_auth + "<<<< \r\n") ;
                    String sAcctDesc;
                    WSResult2OfArrayOfArrayOfString wsgllist = m_WebServ.GetGLAccountList(sAuth);
                    SListAccts.Add("0000-00-000", "No selection") ;
                    dictAccts.Add("0000-00-000",  "No selection") ;
                    if (wsgllist.Status == OperationStatus.Success)
                    {
                        foreach (string[] acct in wsgllist.Result)
                        {
                            // Put accounts in dictionary and combo box - - - - - -
                            dictAccts.Add(acct[0],  acct[1]) ;
                            SListAccts.Add(acct[0], acct[1]) ;
                        } // foreach (string[] acct in wsgllist.Result)
                        //textBoxStat.AppendText("   Retrieved accts: " + dictAccts.Count) ;  //  + "\r\n");
                        textBoxStat.AppendText("   Retrieved accts: " + SListAccts.Count);  //  + "\r\n");
                    } // if (wsgllist.Status == OperationStatus.Success)

                    //  v v v Populate combo boxes with sortedlist of accounts v v v v v v v v v v v v v v v v v v v v v 
                    cbSuspence.Items.Clear();  // <-- Just in case a suspence acct has been plugged in
                    foreach (KeyValuePair<string, string> pair in SListAccts)
                    {
                        // Used Sort List for build Combo Boxes - - - - - - - - - - - -
                        sAcctDesc = String.Format("{0} {1}", pair.Key, pair.Value);
                        cbDebit.Items.Add(sAcctDesc)    ;
                        cbCredit.Items.Add(sAcctDesc)   ;
                        cbSuspence.Items.Add(sAcctDesc) ;
                    }  // foreach (KeyValuePair<string, string> pair in SListAccts)

                    //  v v v Company Name v v v v v v v v v v v v v v v v v v v v v 
                    var compinfo = m_WebServ.GetCompanyInformation(sAuth) ;
                    m_APIkey.CompName = compinfo.Result.CompanyName ;
                    textBoxStat.AppendText(string.Format("\r\nComp name:{0}", m_APIkey.CompName)) ;
                    //textBoxStat.AppendText(string.Format("\r\nAddr:{0}", compinfo.Result.Address)) ;
                    //textBoxStat.AppendText(string.Format("\r\nContact:{0}", compinfo.Result.ContactInformation)) ;
                    //textBoxStat.AppendText(string.Format("\r\nLocale:{0}", compinfo.Result.LocaleName)) ;
                } // if (auth == null)
                fWait.Close() ;
            } 
            catch (ArgumentException eXXX)
            {
                textBoxStat.AppendText("ArgumentException \r\n") ;
                textBoxStat.AppendText("ArgumentMessage:"  + eXXX.Message  + "\r\n ^ ^ ^ ^ ^ ^ \r\n") ;
                textBoxStat.AppendText("ArgumentHelpLink:" + eXXX.HelpLink + "\r\n ^ ^ ^ ^ ^ ^ \r\n") ;
                textBoxStat.AppendText("ArgumentInnerException:" + eXXX.InnerException + "\r\n ^ ^ ^ ^ ^ ^ \r\n");
                textBoxStat.AppendText("ArgumentParamName:" + eXXX.ParamName + "\r\n ^ ^ ^ ^ ^ ^ \r\n");
            }
            catch (IOException eYYY)
            {
                textBoxStat.AppendText("IOException - - - \r\n") ;
                textBoxStat.AppendText("Message:" + eYYY.Message + "\r\n====\r\n") ;
            }
            catch (CommunicationException eZZZ)
            {
                string sCE = eZZZ.Message ;
                textBoxStat.AppendText("CommunicationException \r\n====\r\n");
                textBoxStat.AppendText("Data:" + eZZZ.Data + "\r\n====\r\n") ;
                textBoxStat.AppendText("Message:" + sCE + "\r\n====\r\n") ;
                textBoxStat.AppendText("Target site:" + eZZZ.TargetSite + "\r\n====\r\n") ;
                textBoxStat.AppendText("Help link:"   + eZZZ.HelpLink   + "\r\n====\r\n") ;
                textBoxStat.AppendText("Source:"      + eZZZ.Source     + "\r\n====\r\n") ;
                textBoxStat.AppendText("StackTrace:"  + eZZZ.StackTrace + "\r\n====\r\n") ;
                textBoxStat.AppendText("InnerException:" + eZZZ.InnerException + "\r\n====\r\n") ;
            }
            return bRet; 
        }  // private void AIQ_ChartOfAccts()


        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = 
        private void btnSave_Click(object sender, EventArgs e)
        {
            /*
             *  Save details (Dr/Cr) for selected account
             *  
             */

            int iIdx ; 
            string sAcct ;
            //sSelected = cbCredit.SelectedItem ;
            // Isolate debit account # - - - - - - - - - - - - - - - - 
            if (cbDebit.SelectedIndex >= 0)
            {
                // Selection made on Debit side - - - - - - - - - - - -
                iIdx = cbDebit.SelectedItem.ToString().IndexOf(" ") ;  // <-- Seperate from description
                sAcct = cbDebit.SelectedItem.ToString().Substring(0, iIdx);
                //textBoxStat.AppendText(string.Format("Dr acct:{0} \r\n", sAcct));
                //oLocSel.asDebit[m_SelIndex] = sAcct ;
                if (sAcct.CompareTo("0000-00-000") != 0)  
                    aSetVals[m_SelIndex].DrAcct = sAcct ;
                else
                    aSetVals[m_SelIndex].DrAcct = " "   ;  // <-- User must have selected 0000-00-000 
            }

            // Isolate credit account # - - - - - - - - - - - - - - - - 
            if (cbCredit.SelectedIndex >= 0)
            {
                // Selection made on Credit side - - - - - - - - - - - - 
                iIdx = cbCredit.SelectedItem.ToString().IndexOf(" ");
                sAcct = cbCredit.SelectedItem.ToString().Substring(0, iIdx);
                //textBoxStat.AppendText(string.Format("Cr acct:{0} \r\n", sAcct));
                //aSetVals[m_SelIndex].CrAcct = sAcct ;
                if (sAcct.CompareTo("0000-00-000") != 0)
                    aSetVals[m_SelIndex].CrAcct = sAcct ;
                else
                    aSetVals[m_SelIndex].CrAcct = " "   ;  // <-- User must have selected 0000-00-000 
            }
            //else
            //    textBoxStat.AppendText("Cr blank\r\n");

            // Clear values - - - - - - - - -
            tbDesc.Text = "" ;   
            tbAmt.Text  = "" ;
            tbComment.Text = "" ;
            
            cbDebit.SelectedIndex  = -1 ;   cbCredit.SelectedIndex = -1 ;

            // Disable acct selection controls - - - - - - - - -
            cbDebit.Enabled  = false ;    cbCredit.Enabled = false ;
            btnSave.Enabled  = false ;  

            // Refresh CSV data with accts GL accounts, comments, etc. 
            PopulateCSVListView() ; 

        } // private void btnSave_Click(object sender, EventArgs e)

        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = 
        private void btnPost_Click(object sender, EventArgs e)
        {
            textBoxStat.AppendText("\r\nbtnPost_Click()") ;
            //textBoxStat.AppendText("POST to GL \r\n") ;
            //bool bSusUsed = false ; 
            int iTransCnt = 0;
            decimal dcBalance = 0; 
            string sSusAcct = " " ;   
            if ( listTran.Count > 0 )   //   ( m_AcctAssignCnt > 0 )
            {
                if (m_Balance == 0 || 
                    (cboxSuspence.Checked && 
                     cbSuspence.SelectedIndex >= 0 && 
                    !cbSuspence.SelectedItem.ToString().StartsWith("0000-00")) )
                {
                    foreach (TranStruc oTran in listTran)
                    {
                        iTransCnt++ ;
                        dcBalance += oTran.Amt ;  // <-- Recalc balance just to be sure 
                        //textBoxStat.AppendText(string.Format("\r\n post {0} {1:0.00} d:{2}<  ",
                        //                    oTran.Acct, oTran.Amt, oTran.Desc));
                    }
                    textBoxStat.AppendText(string.Format("\r\n Tran lines:{0} balance:{1:0.00} ",
                                        iTransCnt, dcBalance));

                    m_SusAmt = dcBalance * -1 ;  // <-- 10 May '17 Keep amount for posting to API log 
                    if (dcBalance != 0)  // <-- Are the tranactions out of balance? 
                    {
                        // Add entry for suspence acct, if needed - - - - - - - - - - 
                        int iPos = cbSuspence.SelectedItem.ToString().IndexOf(" ") ;
                        if (iPos > 0)  // <-- true if we have acct with description
                            sSusAcct = cbSuspence.SelectedItem.ToString().Substring(0, iPos) ;
                        else
                            sSusAcct = cbSuspence.SelectedItem.ToString() ;
                        TranStruc JLine ;
                        JLine.Acct = sSusAcct ;
                        JLine.Amt  = dcBalance * -1 ;
                        JLine.Desc = sSetLocCode + " " + sCurrDateTime ;  //  " sus";
                        listTran.Add(JLine) ;
                        //bool bSusUsed = true ;
                    } // if (dcBalance != 0)

                    //textBoxStat.AppendText(string.Format("\r\nAIQ_post() cnt:{0}", AIQ_Post()) );
                    //AIQ_Post();
                    int iJournalLines = AIQ_Post() ;
                    if (iJournalLines > 0)
                    {
                        SaveMappedAccts(sSusAcct) ;
                        LogAPIpost(iJournalLines) ; 
                        listTran.Clear();  // <-- 8 May '17  Clear this after successful post 
                    } // if (AIQ_Post() > 0)
                    else
                    {
                        LogAPIpost(iJournalLines) ;
                    }

                    //this.Close();  

                }  // if (m_Balance == 0 || (cboxSuspence.Checked && cbSuspence.SelectedIndex > 1))
                else
                    textBoxStat.AppendText("\r\nOut of balance and no sus ");
            }  //  if (m_AcctAssignCnt > 0)
            else
                textBoxStat.AppendText("\r\nlistTran.Count <= 0 ") ;
        } // private void btnPost_Click(object sender, EventArgs e)


        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = 
        private int AIQ_Post()
        {
            /*
             * Actually execute post to AIQ GL 
             * 
             */

            textBoxStat.AppendText("\r\nAIQ_post() ");

            int iRet = -1 ;  // Return journal lines posted
            sCurrDateTime = DateTime.Now.ToString(@"M\/d H\:mm");  // DateTime.Now.ToString("MM/dd h:mm tt")

            // 9 May '17 - Better log in regardless. Sometimes the "token" can expire
            // We don't have a dictionary of accounts so we must haven't logged in yet
            //m_auth = m_WebServ.Login(m_sCompID, m_keyPart, m_keyUser);
            string sAuth = m_WebServ.Login(m_APIkey.CompID, m_APIkey.keyPart, m_APIkey.keyUser);  // m_sCompID, m_keyPart, m_keyUser);
            if (sAuth == null)
            {
                textBoxStat.AppendText("\r\nAIQ Login failed ") ;
                return -2 ; 
            }

            // Actual code for AIQ API - - - - - - - - - - - - -
            GeneralJournal journal = new GeneralJournal
            {
                ExternalReference = "JV-TEST",
                InternalReference = string.Format("Loc:{0} {1}", sSetLocCode, sCurrDateTime),
                TransactionDate = oSetDate
            } ;

            // 6 May 2017 - My hack to create API transaction to post 
            int iLnSub = 0;  // <-- Subscript for journal.Lines array 
            journal.Lines = new GeneralJournalLine[listTran.Count];
            foreach (TranStruc oTran in listTran)
            {
                //iTransCnt++;
                //dcBalance += oTran.Amt;
                //textBoxStat.AppendText(string.Format("\r\n AIQ_post {0} {1:0.00} d:{2}<  ",
                //                    oTran.Acct, oTran.Amt, oTran.Desc));
                GeneralJournalLine SingleLine = new GeneralJournalLine();
                SingleLine.GLAccountCode = oTran.Acct ;
                SingleLine.Amount        = oTran.Amt  ;
                SingleLine.Description   = oTran.Desc ;
                journal.Lines[iLnSub] = SingleLine ;
                iLnSub++;
            } // foreach (TranStruc oTran in listTran)

            WSResultStatus GLstat = m_WebServ.CreateGeneralJournal(sAuth, journal);
            if (GLstat.Status == OperationStatus.Success)
            {
                textBoxStat.AppendText("\r\nAIQ GL post successful, YEA!! = = = ");
                iRet = iLnSub    ;  // Return number of transaction lines posted 
            }
            else
            {
                textBoxStat.AppendText("\r\nGL post failed, darn = = = ");
                textBoxStat.AppendText(string.Format("\r\nErrorCode:{0}", GLstat.ErrorCode));
                textBoxStat.AppendText(string.Format("\r\nErrorMessage:{0}", GLstat.ErrorMessage));
                textBoxStat.AppendText(string.Format("\r\nStatus:{0}", GLstat.Status));
            }

            return iRet ; 
        } //private void AIQ_post(float inAmt, string inDrAcct, string inCrAcct) 


        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = 
        private int SaveMappedAccts(string inSusAcct)
        {
            int iRet = 0 ;

            m_conn.Open() ;
            OleDbCommand cmdSel = new OleDbCommand();
            cmdSel.Connection = m_conn;
            for (int iLnSub = 0; iLnSub < SetSize; iLnSub++)
            {
                if ( aSetVals[iLnSub].DrAcct.CompareTo(" ") > 0 
                  || aSetVals[iLnSub].CrAcct.CompareTo(" ") > 0
                  || aSetVals[iLnSub].InMDB )  // <-- Try to catch items that have been "unmapped"
                {
                    //if (!aSetVals[iLnSub].InMDB)
                    //{
                    iRet++;
                    cmdSel.CommandText = string.Format("DELETE from AcctMap WHERE Location = '{0}' AND  FullDesc = '{1}'", 
                                                            sSetLocCode, aSetVals[iLnSub].Desc) ;
                    cmdSel.ExecuteNonQuery() ;

                    if (aSetVals[iLnSub].DrAcct.CompareTo(" ") > 0 || aSetVals[iLnSub].CrAcct.CompareTo(" ") > 0)
                    {
                        //cmdSel.CommandText = string.Format("Insert into AcctMap (Location, FullDesc, DrSide, CrSide) values ('{0}', '{1}', '{2}', '{3}')",
                        cmdSel.CommandText = "INSERT into AcctMap (Location, FullDesc, DrSide, CrSide)";
                        cmdSel.CommandText += string.Format(" values ('{0}', '{1}', '{2}', '{3}')",
                                                        sSetLocCode, aSetVals[iLnSub].Desc,
                                                        aSetVals[iLnSub].DrAcct, aSetVals[iLnSub].CrAcct);
                        cmdSel.ExecuteNonQuery();
                    }
                    //}
                    /*
                    else
                    {
                        cmdSel.CommandText = string.Format("UPDATE AcctMap SET DrSide = '{0}', DrSide = '{1}'",
                                                        aSetVals[iLnSub].DrAcct, aSetVals[iLnSub].CrAcct);
                        cmdSel.CommandText += string.Format(" WHERE Location = '{0}' AND  FullDesc = '{1}' ;",
                                                        sSetLocCode, aSetVals[iLnSub].Desc);
                        cmdSel.ExecuteNonQuery();
                    }*/
                } // if (aSetVals[iLnSub].DrAcct.CompareTo(" ") > 0 || aSetVals[iLnSub].CrAcct.CompareTo(" "))
            } //  for (int iLnSub = 0; iLnSub < SetSize; iLnSub++)

            if (inSusAcct.CompareTo(" ") > 0)
            {
                iRet++;
                cmdSel.CommandText = string.Format("DELETE from AcctMap WHERE Location = '{0}' AND  FullDesc = '{1}'",
                                                    sSetLocCode, "~SUSPENCE") ;
                cmdSel.ExecuteNonQuery();

                cmdSel.CommandText = "INSERT into AcctMap (Location, FullDesc, DrSide)";
                cmdSel.CommandText += string.Format(" values ('{0}', '~SUSPENCE', '{1}')",
                                                 sSetLocCode, inSusAcct) ;
                cmdSel.ExecuteNonQuery();

                /*
                if (!m_bSusAcctRead)
                {
                    cmdSel.CommandText = "INSERT into AcctMap (Location, FullDesc, DrSide)";
                    cmdSel.CommandText += string.Format(" values ('{0}', '~SUSPENCE', '{1}')",
                                                     sSetLocCode, inSusAcct);
                    cmdSel.ExecuteNonQuery();
                }
                else
                {
                    cmdSel.CommandText = string.Format("UPDATE AcctMap SET DrSide = '{0}'", inSusAcct);
                    cmdSel.CommandText += string.Format(" WHERE Location = '{0}' AND  FullDesc = '~SUSPENCE' ;",
                                                    sSetLocCode);
                    cmdSel.ExecuteNonQuery();
                }  */
            } // if ( sSusAcct.CompareTo(" ") != 0 )

            return iRet ;

        }  // private int SaveMappedAccts(string inSusAcct)


        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = 
        private int LogAPIpost(int inLineCount)
        {
            /*
             *   Log successful API post to AIQ.
             *   
             */

            int iRet = 0 ;
            OleDbConnection OLEconn = new OleDbConnection(m_connStr);

            string LogCode = sSetLocCode ;
            if (Post2Test) 
                LogCode += " (test)" ;
            try
            {
                OLEconn.Open() ;

                OleDbCommand cmdSel = new OleDbCommand() ;
                cmdSel.Connection = OLEconn ;

                String dtNow = DateTime.Now.ToString() ;
                cmdSel.CommandText = "Insert into APIpost (DTstamp, Location, BusDate, JLines, SusAmt)" ;
                cmdSel.CommandText += string.Format(" values ('{0}', '{1}', '{2}', '{3}', {4});",
                                                       dtNow, LogCode, sSetDate,
                                                       listTran.Count, m_SusAmt.ToString() ) ;
                cmdSel.ExecuteNonQuery() ;
                iRet = 1 ;
                //textBoxStat.AppendText(string.Format("\r\nLogAPIpost() post has been logged >{0}",
                //                                            cmdSel.CommandText ) ) ;
            
            }
            catch (Exception ex)
            {
                iRet = -1;
                textBoxStat.AppendText("\r\nLogAPIpost() open failed") ;
                textBoxStat.AppendText(string.Format("\r\nerror:{0}",ex.Message));
            }
            finally
            {
                OLEconn.Close() ;
            }

            return iRet ;
        } //  private int LogAPIpost(int inLineCount)

        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = 
        private void cboxSuspence_CheckedChanged(object sender, EventArgs e)
        {
            int iSusCnt ; 
            if (cboxSuspence.Checked)
            {
                textBoxStat.AppendText("\r\nBox is checked.");
                iSusCnt = cbSuspence.Items.Count ;
                textBoxStat.AppendText(string.Format("  cnt:{0}", iSusCnt));
                if (iSusCnt == 0)
                {
                    // Combo Box is empty - lets add the suspence acct, if there is one already mapped - - -
                    if (m_sSusAcct.CompareTo(" ") > 0)
                    {
                        cbSuspence.Items.Add(m_sSusAcct);
                        cbSuspence.SelectedIndex = 0;
                    }
                    /*
                    else
                    {
                        if (SListAccts.Count < 10)  // 10 is arbitrary 
                        {
                            // We must not have the chart of accounts , so get them 
                            AIQ_ChartOfAccts_etc() ; 
                        }
                    }
                    */
                }
                else if (iSusCnt == 1)
                {
                    // There is just one item so set index to it.
                    cbSuspence.SelectedIndex = 0;
                }
                //else
                //{
                //    if SListAccts.Count     //cbDebit.SelectedIndex = SListAccts.IndexOfKey(sAcct) ; 
                //}
                cbSuspence.Enabled = true;
                cbSuspence.DropDownStyle = ComboBoxStyle.DropDownList;
            }
            else
            {
                //textBoxStat.AppendText("\r\nBox is unchecked.");
                cbSuspence.Enabled = false ;
            }
            ChkButtonPost() ;
        }  // private void cboxSuspence_CheckedChanged(object sender, EventArgs e)


        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = 
        private void cbSuspence_SelectedIndexChanged(object sender, EventArgs e)
        {
            //if (cbSuspence.SelectedIndex >= 1)
            ChkButtonPost() ; 
        }

        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = 
        private void listViewAccts_SelectedIndexChanged(object sender, EventArgs e)
        {
            //textBoxStat.AppendText("SelectedIndexChanged\r\n");
            //textBoxStat.AppendText("sender:" + sender.ToString() ) ;
            //textBoxStat.AppendText("e:" + e.ToString()) ;
        }

        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = 
        private void textBoxStat_TextChanged(object sender, EventArgs e)
        {
            //textBoxStat.AppendText("sender:" + sender.ToString() );
            //textBoxStat.AppendText("e:" + e.ToString());
        }


        private void btnCancel_Click(object sender, EventArgs e)
        {
            this.Close(); 
        }


        private void textBox1_TextChanged(object sender, EventArgs e)
        {

        }
    } // public partial class ImportForm2 : Form
}

/*
 * 16 Mar '17 
 * 
 * SOAP keep having CommunicationExceptions. I forgot about the buffer size issue 
 * from Tutorial_A  (SOAP test).
 * 
 * Below are changed I made in order until AIQ SOAP log in worked.
 * 
 * 
 * <binding name="Integration_1_1Soap1" 
 * maxBufferSize="965536" ...doesn't help
 * maxBufferPoolSize="9524288" ...doesn't help
 * maxReceivedMessageSize="965536" ...doesn't help
 * readerQuotas maxDepth="9932"  ...doesn't help
 * maxStringContentLength="98192" ...doesn't help
 * maxArrayLength="996384"  ...doesn't help
 * maxBytesPerRead="94096"
 * maxNameTableCharCount="916384" 
 * 
 * binding name="Integration_1_1Soap"
 * maxBufferSize="9965536"
 *   ArgumentException 
 *   ArgumentMessage:For TransferMode.Buffered, MaxReceivedMessageSize and MaxBufferSize must be the same value.
 *   Parameter name: bindingElement
 * maxReceivedMessageSize="9965536"
 * maxBufferPoolSize="9524288"  ...CommunicationException 
 * maxDepth="9932"  ...CommunicationException 
 * maxStringContentLength="98192"  ...CommunicationException 
 * maxArrayLength="916384"  ...CommunicationException 
 * maxBytesPerRead="94096"  ...CommunicationException 
 * maxNameTableCharCount="916384"  ...Auth not null   (worked) 
 * 
 */


/*
journal.Lines = new GeneralJournalLine[]
{
    new GeneralJournalLine { GLAccountCode = "1050-00-000", Amount = 330.0M,  Description = sDrCrDesc },   // "Sales posted by JV(POS)" } ,
    //new GeneralJournalLine { GLAccountCode = inDrAcct, Amount = dDrAmt, Description = sDrCrDesc } ,
    new GeneralJournalLine { GLAccountCode = "1105-00-000", Amount = -330.0M, Description = sDrCrDesc }   // "Sales posted by JV(POS)" },
    //new GeneralJournalLine { GLAccountCode = inCrAcct, Amount = dCrAmt, Description = sDrCrDesc }
} ;  */

```
