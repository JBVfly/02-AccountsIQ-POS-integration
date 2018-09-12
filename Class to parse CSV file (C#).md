

``` C#

using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace POS_Int
{
    class FileCSV
    {
       /*
        *   There should be one instance of this class for each CSV file
        *   
        */

        private enum LexState { StartLine, StartToken, Char, Numeric, Neg, Dash, AlphaNum, Date, Space } ;
        private enum CharType { DQuote, Dollar, Comma, Char, Dig, Space, Slash, Dash, LParen, RParen, Decimal, Other } ;
        private enum ExpState { None, Report, Net_Rev } ;

        // Collect text for debugging
        public String m_Debug { get; set; }
        private StringBuilder m_DebugSB = new StringBuilder();

        // Values to passback = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = =
        public String m_ReportCode { get; set; }
        //public String m_ReportLine { get; set; } 
        public String m_Location { get; set; }
        public bool CSVisValid { get; set; }
        public int m_iLnCnt { get; set; }
        public int iAmtCount { get; set; }
        public String[] aDesc = new String[100];
        public String[] asAmt = new String[100];
        //public float[] afAmt = new float[100];
        public decimal[] adecAmt = new decimal[100] ;  // 5 VII '17  - Change to decimal from float
        public String[] asDebit = new String[100];
        public String[] asCredit = new String[100];
        public String[] asRptLn = new String[100]; // <-- 7 May '17 Report/line source of amt 
        public String[] asComment = new String[100];
        // 7 May '17  use a structure for each amt. Surprised I didn't do this earlier 
        public struct DescAmtStruc
        {
            public string Desc;
            //public string sAmt;  // 5 VII '17 Remmed out since it appeared to be used
            //public float fAmt;
            //public decimal decAmt;  // 5 VII '17  - Change to decimal from float
            public string RptLn;
            //public string Debit, Credit ;
        }
        public DescAmtStruc[] aCSVDescAmt = new DescAmtStruc[100];

        public DateTime oReportDate { get; set; }
        public String sReportDate { get; set; }

        public string SortKey { get; set; }

        public string ShortFileName { get; set; }

        private int m_iAmtSub = 0;

        //private float m_fNetRev;
        private decimal m_decNetRev;  // 5 VII '17  - Change to decimal from float
        private String sFile;

        private String[,] aCSVtokens = new String[100, 100];
        private int m_iLnSub;

        private int m_iLastStaticSub;

        //private int m_InvalidCnt ;   // <-- 1 May '17 Count of invalid files

        private struct TokStruc
        {
            public string TDesc;
            public int TCnt;
        }
        private Dictionary<string, TokStruc> dictToks = new Dictionary<string, TokStruc>();

        private struct SegStruc
        {
            public string TVal;
            public string TAttr;
        }

        private SegStruc[,] aSegments = new SegStruc[100, 100];

        private Dictionary<string, int> dictColumns = new Dictionary<string, int>();

        public FileCSV(String sIn)
        {
            /*
             *  Set file name
             *  
             */
            CSVisValid = false;  // <-- File not valid until proven so. 
            sReportDate = "(no Date)";
            oReportDate = Convert.ToDateTime("1/1/2000");
            m_Location = "CSV file doesn't conform to POS export format";
            sFile = sIn;
        }

        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = =
        public int ReadCSV()
        {
            string sSeg, sAttr, sNum;
            String line;  // <-- Line read in 
            m_Debug = "";
            m_DebugSB.Append("\r\n   Start ReadCSV...");
            m_ReportCode = "(none)";
            System.IO.StreamReader file =             //  <-- Open CSV file 
               new System.IO.StreamReader(sFile);    //  

            m_DebugSB.Append(string.Format("\r\n   Read file:{0} = = = = = = = = = = = = = = = = = = = =", sFile));
            m_iLnSub = 0;
            m_iLastStaticSub = 0;
            // Loop through every line = = = = = = = = = = = = = = = = = = = = = = = = 
            while ((line = file.ReadLine()) != null)
            {
                ProcessLine2(line);  // <-- 13 Apr '17  New, Improved parcing 
                m_iLnSub++;
                if (m_iLnSub >= 98)  // Limit lines for testing,  98 for catching non-conforming CSV files 
                    break;
            } // while ((line = file.ReadLine()) != null) 
            file.Close(); file.Dispose();
            //m_Debug += string.Format("\r\nm_iLnSub:{0}", m_iLnSub) ;
            m_DebugSB.Append(string.Format("\r\n   m_iLnSub:{0}", m_iLnSub));
            m_iLnCnt = m_iLnSub;

            bool bValidDate = false, bValidLocation = false, bValidReport = false;

            m_iLastStaticSub = -1;
            ExpState ExpTok = ExpState.None;
            foreach (KeyValuePair<string, TokStruc> pair in dictToks)
            {
                //m_Debug += string.Format("\r\n{0}|{1}   cnt:{2}",
                //    pair.Key, pair.Value.TDesc, pair.Value.TCnt.ToString());
                //m_DebugSB.Append(string.Format("\r\n   {0}|{1}   cnt:{2}",
                //    pair.Key, pair.Value.TDesc, pair.Value.TCnt.ToString())) ; 
                if (pair.Value.TCnt == m_iLnSub)  // <-- Look at items that appear in all lines
                {
                    //m_Debug += string.Format("\r\n{0}|{1}   cnt:{2}",
                    //    pair.Key, pair.Value.TDesc, pair.Value.TCnt.ToString());
                    m_iLastStaticSub++;
                    sNum = pair.Key.Substring(0, 3);
                    sSeg = pair.Key.Substring(4);
                    sAttr = pair.Value.TDesc;
                    switch (ExpTok)
                    {
                        case ExpState.Report:
                            if (sAttr.CompareTo("C") == 0)
                            {
                                m_ReportCode = sSeg;  // <-- Found report code 
                                m_Debug += "   report found";
                                bValidReport = true;
                            }
                            ExpTok = ExpState.None;
                            break;

                        case ExpState.Net_Rev:
                            if (sAttr.CompareTo("N") == 0)
                            {
                                //m_decNetRev = FloatSeg(sSeg);
                                m_decNetRev = DecimalSeg(sSeg) ;
                                m_DebugSB.Append(string.Format("\r\n   Set NetRev:{0}", m_decNetRev));
                            }
                            ExpTok = ExpState.None;
                            break;

                        default:
                            if (sSeg.StartsWith("REPORT: ") && sAttr.CompareTo("CC") == 0)
                            {
                                m_Debug += " looking for report";
                                m_ReportCode = sSeg.Substring(8);
                                bValidReport = true;
                            }
                            else if (sSeg.CompareTo("REPORT:") == 0 && sAttr.CompareTo("C") == 0)
                            {
                                ExpTok = ExpState.Report;
                                m_Debug += "  look in next seg";

                            }
                            else if (sAttr.StartsWith("CCCCCC"))
                            {
                                if (sSeg.Substring(5, 3).CompareTo(" - ") == 0 && m_Location.StartsWith("CSV file doesn't"))
                                {
                                    m_Location = sSeg;
                                    bValidLocation = true;
                                }
                            }
                            else if (sSeg.StartsWith("Report for ") && sAttr.CompareTo("CCD") == 0)
                            {
                                sReportDate = sSeg.Substring(11);
                                oReportDate = Convert.ToDateTime(sReportDate);
                                m_Debug += "   date found";
                                bValidDate = true;
                            }
                            else if (sSeg.StartsWith("FOR ") && sAttr.CompareTo("CD") == 0)
                            {
                                sReportDate = sSeg.Substring(4);
                                oReportDate = Convert.ToDateTime(sReportDate);
                                m_Debug += "   date found";
                                bValidDate = true;
                            }
                            else if (sSeg.StartsWith("ROOM/GUEST STATISTICS FOR: ") && sAttr.CompareTo("CCCD") == 0)
                            {
                                sReportDate = sSeg.Substring(26);
                                oReportDate = Convert.ToDateTime(sReportDate);
                                m_Debug += "  date found";
                                bValidDate = true;
                            }
                            else if (sSeg.CompareTo("NET ROOM/SUITE REVENUE") == 0)
                            {
                                ExpTok = ExpState.Net_Rev;
                                m_DebugSB.Append(" looking for Net Rev");

                            }
                            break;
                    } // switch (ExpTok)
                }  // if (pair.Value.TCnt == m_iLnSub)
                //m_iLastStaticSub = Int32.Parse(sNum);
            } // foreach (KeyValuePair<string, TokStruc> pair in dictToks
            //m_Debug += string.Format("\r\nm_iLastStaticSub:{0}", m_iLastStaticSub) ;
            m_DebugSB.Append(string.Format("\r\n   m_iLastStaticSub:{0}", m_iLastStaticSub));

            if (bValidDate && bValidLocation && bValidReport)
            {
                //m_Debug += "\r\nThis CSV is valid" ;
                m_DebugSB.Append("\r\n   This CSV is valid");
                CSVisValid = true;
            }

            // Process appropriate report - - - - - - - - - - - - - - - - - - - - - - - - - -
            switch (m_ReportCode)
            {
                case "RVYACCTG": ProcessRVYACCTG(); break;
                case "RECRCP": ProcessRECRCP(); break;
                case "GSTSTATS": ProcessGSTSTATS(); break;

            } // switch (m_Report)

            iAmtCount = m_iAmtSub;  // Total number of non-zero lines - - - - - -
            //m_Debug += string.Format("\r\niAmtCount:{0}", iAmtCount);
            m_DebugSB.Append(string.Format("\r\n   iAmtCount:{0}", iAmtCount));

            ShortFileName = GetFileName();

            m_Debug = m_DebugSB.ToString();
            SortKey = "00000000AAAAA";
            if (CSVisValid)
                SortKey = string.Format("{0}{1}{2}/{3}", oReportDate.ToString("yyyyMMdd"),
                                                     m_Location.Substring(0, 5),
                                                     m_ReportCode,
                                                     ShortFileName);
            else
            {
                //m_InvalidCnt++ ;
                //SortKey = string.Format("00000000AA{0}/{1}", m_InvalidCnt.ToString("000"), ShortFileName) ;
                //SortKey = string.Format("00000000AA/{0}", ShortFileName) ;
                SortKey = string.Format("00000000AA{0}", ShortFileName);
            }

            return m_iLnCnt;
        } // public int ReadCSV() - - - -


        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = =
        private int ProcessLine2(string inLine)
        {
            /*
             *  Process each line
             * 
             */
            int iRet = 0;
            int iColSub = 0;
            //string sFinalToken ; 
            char[] aChar = inLine.ToCharArray();
            LexState LState;
            CharType CType;

            //m_Debug += string.Format("\r\nStart ProcessLine2() with {0} characters", aChar.Length);

            bool bInQuote = false;
            StringBuilder sb = new StringBuilder();
            //LState = LexState.StartLine;
            LState = LexState.StartToken;
            string sTokTypes = "";

            string sAppend = "";

            foreach (char cChar in aChar)
            {
                CType = ClassChar(cChar);
                sAppend = string.Format("\r\nChar: {0} type:{1} stat:{2}", cChar, CType, LState);
                switch (CType)
                {
                    case CharType.DQuote:
                        bInQuote = (!bInQuote);
                        if (bInQuote)
                            LState = LexState.StartToken;
                        break;

                    case CharType.Comma:
                        if (bInQuote)
                        {
                            if (LState != LexState.Numeric && LState != LexState.Neg)  // <-- Drop commas in numbers
                                sb.Append(cChar);
                        }
                        else
                        {
                            // Hit comma not within double-quotes - - - - - - - - - - - - - - - -
                            sTokTypes += AssignTokCode(LState);
                            // We are moving into a new token - - - - - - - - - - - - - - - - - -
                            RecordToken2(sb.ToString(), sTokTypes, iColSub);
                            sb = new StringBuilder();
                            sTokTypes = "";
                            LState = LexState.StartToken;
                            iRet++;
                            iColSub++;
                        }
                        break;

                    case CharType.Char:  // A to Z  (or a to z) 
                        switch (LState)
                        {
                            case LexState.Numeric:
                                LState = LexState.AlphaNum;
                                break;

                            default:
                                LState = LexState.Char;
                                break;
                        } // switch (LState)
                        sb.Append(cChar);
                        break;

                    case CharType.Space: // Space 
                        if (LState != LexState.StartToken)
                        {
                            // This should exclude leading space in the token - - -
                            sTokTypes += AssignTokCode(LState);
                            LState = LexState.Space;
                            sb.Append(cChar);
                        }
                        break;

                    case CharType.Dig:  // 1 to 9
                        switch (LState)
                        {
                            //case LexState.StartLine:
                            case LexState.StartToken:
                            case LexState.Space:
                                LState = LexState.Numeric;
                                break;

                            case LexState.Dash:
                                LState = LexState.Neg;
                                break;

                            case LexState.Char:
                                LState = LexState.AlphaNum;
                                break;
                                //default:
                                //    LState = LexState.Numeric ;
                                //    break ;
                        } // switch (LState)
                        sb.Append(cChar);
                        break;

                    case CharType.LParen:  // "("
                        switch (LState)
                        {
                            case LexState.StartLine:
                            case LexState.StartToken:
                            case LexState.Space:
                                LState = LexState.Neg;
                                break;

                        } // switch (LState)
                        sb.Append(cChar);
                        break;

                    case CharType.Dollar:  // "$"
                        switch (LState)
                        {
                            case LexState.StartLine:
                            case LexState.StartToken:
                            case LexState.Space:
                            case LexState.Neg:
                                //LState = LexState.Neg;
                                break;
                            default:
                                sb.Append(cChar);
                                break;

                        } // switch (LState)
                        //sb.Append(cChar);
                        break;

                    case CharType.Dash:  // "-"
                        switch (LState)
                        {
                            case LexState.StartLine:
                            case LexState.StartToken:
                            case LexState.Space:
                                LState = LexState.Dash;
                                break;

                            //case LexState.Neg:
                            default:
                                LState = LexState.Char;
                                break;

                        } // switch (LState)
                        sb.Append(cChar);
                        break;

                    case CharType.Decimal:  // "."
                        sb.Append(cChar);
                        break;

                    case CharType.RParen:   // ")"
                        sb.Append(cChar);
                        break;

                    case CharType.Slash:    // "/" 
                        switch (LState)
                        {
                            case LexState.Numeric:
                                LState = LexState.Date;
                                break;

                        } // switch (LState)
                        sb.Append(cChar);
                        break;

                    default:
                        sb.Append(cChar);
                        break;
                } //  switch (cChar) 
                sAppend += string.Format("->{0}", LState);
            }  // foreach (char cChar in aChar)

            sTokTypes += AssignTokCode(LState);
            RecordToken2(sb.ToString(), sTokTypes, iColSub); // <-- Get the very last token  

            iRet++;
            return iRet;
        }  // private int ProcessLine2(string inLine)


        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = =
        private void RecordToken2(string inSegment, string inAttr, int inCol)
        {
            KeyIncr2(inSegment, inAttr, inCol);
            aCSVtokens[m_iLnSub, inCol] = inSegment;

            SegStruc SData = new SegStruc();
            SData.TVal = inSegment;
            SData.TAttr = inAttr;
            if (inCol <= 99)  // <-- Catch lines with too many columns
                aSegments[m_iLnSub, inCol] = SData;
        } // private void RecordToken2(string inSegment, string inAttr, int inCol)


        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = =
        private int KeyIncr2(string inKey, string inAttr, int inCol)
        {
            /*
             *  Build a count of recurrent tokens
             *  
             */
            string sKey = inCol.ToString("000") + ":" + inKey;

            TokStruc TData = new TokStruc();
            TData.TDesc = inAttr;

            if (!dictToks.ContainsKey(sKey))
            {
                TData.TCnt = 1;
                dictToks.Add(sKey, TData);
            }
            else
            {
                TData = dictToks[sKey];
                TData.TCnt++;
                dictToks[sKey] = TData;
            }

            SegStruc SData = new SegStruc();
            SData.TVal = inKey;
            SData.TAttr = inAttr;
            if (inCol <= 99)  // <-- Catch lines with too many columns
                aSegments[m_iLnSub, inCol] = SData;

            return TData.TCnt;  // dictToks dictColumns[sKey];
        }  // private int KeyIncr2(string inKey, string inDesc, int inCol)


        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = =
        private void ProcessRVYACCTG()
        {
            /*
             * Process PTD/YTD Accountng Report
             * 
             * 
             */
            SegStruc SData;
            int iSub2;

            //float fAmt = 99;
            decimal decAmt = 99 ;
            string sDesc;
            StringBuilder sb = new StringBuilder();
            for (int iSub1 = 0; iSub1 < m_iLnSub; iSub1++)  // <-- Process all rows < < < < < < < < < < < < 
            {
                //m_Debug += string.Format("\r\n>>>aSeg-{0}", iSub1);
                m_DebugSB.Append(string.Format("\r\n   >>>aSeg-{0}", iSub1));
                iSub2 = m_iLastStaticSub;
                sb = new StringBuilder();
                //fAmt = 999.87f;
                decAmt = 999.87m ;
                while (true) // <-- Process applicable columns  < < < < < < < < < < < <
                {
                    iSub2++;
                    SData = aSegments[iSub1, iSub2];
                    //m_Debug += string.Format(":{0}#{1} ({2})", iSub2, SData.TVal, SData.TAttr) ;
                    if (SData.TAttr.CompareTo("N") == 0 || SData.TAttr.CompareTo("G") == 0)
                    {
                        //fAmt = FloatSeg(SData.TVal);
                        decAmt =  DecimalSeg(SData.TVal);
                        //m_Debug += string.Format(" float val:{0}", FloatSeg(SData.TVal) ) ;
                        break;
                    }
                    if (SData.TVal.CompareTo("END OF REPORT") == 0)
                    {
                        decAmt = 999.77m; break;
                    }
                    if (sb.Length > 1) sb.Append(">" + SData.TVal); else sb.Append(SData.TVal);
                } // while (true)
                sDesc = sb.ToString();
                int iDispLn;
                string sRptLn;
                if (decAmt != 0)
                {
                    // Build list of non-zero items 
                    //m_Debug += string.Format("\r\nDesc build:{0}<< ${1}", sDesc, fAmt) ;
                    m_DebugSB.Append(string.Format("\r\n   Desc build:{0}<< ${1}", sDesc, decAmt));
                    iDispLn = iSub1 + 1;
                    sRptLn = string.Format("{0}:{1}", m_ReportCode, iDispLn.ToString("000"));
                    //aDesc[m_iAmtSub] = sDesc + " [" + m_ReportCode + ":" + iDisp.ToString("000") + "]";
                    aDesc[m_iAmtSub] = string.Format("{0} [{1}]", sDesc, sRptLn);    // +" [" + m_ReportCode + ":" + iDisp.ToString("000") + "]";
                    //afAmt[m_iAmtSub] = fAmt;
                    adecAmt[m_iAmtSub] = decAmt ;
                    asAmt[m_iAmtSub] = decAmt.ToString("F");
                    asDebit[m_iAmtSub] = " ";
                    asCredit[m_iAmtSub] = " ";
                    aCSVDescAmt[m_iAmtSub].Desc = sDesc;   // <--  7 May '17 Starting using struc for passback
                    aCSVDescAmt[m_iAmtSub].RptLn = sRptLn;   // <--
                    m_DebugSB.Append(string.Format("\r\n  sRptLn:{0}< ", sRptLn));   // <-- 7 May 2017
                    m_DebugSB.Append(string.Format("  aCSVDescAmt[m_iAmtSub].RptLn:{0}< ", aCSVDescAmt[m_iAmtSub].RptLn));   // <-- 7 May 2017
                    m_iAmtSub++;
                }  //  if (fAmt != 0)
            } //  for (int iSub1 = 0; iSub1 < m_iLnSub; iSub1++)
        }  // private void ProcessRVYACCTG()

        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = =
        private void ProcessRECRCP()
        {
            /*
             *  Process Receivable Recap
             * 
             */

            //m_Debug += "\r\nProcessRECRCP()";
            m_DebugSB.Append("\r\n   ProcessRECRCP()");

            SegStruc SData;
            int iSub2, iPos;

            //float fAmt = 99, fSegVal;
            decimal decAmt = 99m, decSegVal ;
            string sDesc = "", sBlock = "";
            bool bInNums;
            StringBuilder sb = new StringBuilder();
            for (int iSub1 = 0; iSub1 < m_iLnSub; iSub1++)  // <-- Process all rows < < < < < < < < < < < < 
            {
                //m_Debug += string.Format("\r\n>>>aSeg:Line {0}", iSub1+1);
                sb = new StringBuilder();
                decAmt = 987.87m;
                iSub2 = m_iLastStaticSub;
                bInNums = false;  // <-- 20 Apr '17 moved up a few lines to solve error.
                while (true) // <-- Process applicable columns  < < < < < < < < < < < <
                {
                    iSub2++;
                    SData = aSegments[iSub1, iSub2];
                    //m_Debug += string.Format(" |{0}#>{1}({2})", iSub2, SData.TVal, SData.TAttr);
                    if (SData.TAttr.CompareTo("N") == 0 || SData.TAttr.CompareTo("G") == 0)
                    {
                        bInNums = true; // Move into an area with numbers - - - -
                        //fSegVal = FloatSeg(SData.TVal);
                        decSegVal = DecimalSeg(SData.TVal);
                        decAmt = decSegVal ; 
                        if (decSegVal != 0 && decSegVal != 999999.88m)
                        {
                            //m_Debug += string.Format(" Found {0} drop out", FloatSeg(SData.TVal) ) ;
                            break;
                        }
                    }
                    //m_Debug += string.Format(" {0}", bInNums) ;
                    if (SData.TVal.CompareTo("END OF REPORT") == 0)
                    {
                        //fAmt = 0.00f;
                        //m_Debug += " (EoR)" ;
                        break;
                    }
                    if (SData.TAttr.StartsWith("C"))
                    {
                        if (bInNums)
                        {
                            //m_Debug += "\r\nPassed nums" ;
                            break;
                        }

                        if (iSub2 == m_iLastStaticSub + 2)
                        {
                            //m_Debug += " (We are in 2nd col with C) " ;
                            iPos = sDesc.IndexOf('>');
                            if (iPos > 1)
                            {
                                sBlock = sDesc.Substring(iPos + 1); // <-- Seperate old block from new block
                                //m_Debug += string.Format(" (strip out new block:{0} ", sBlock) ;
                            }
                            else
                            {
                                sBlock = sDesc;    // <-- No old block to seperate 
                                //m_Debug += string.Format(" (first block:{0} ", sBlock) ;
                            }
                            sDesc = string.Format("{0}>{1}", sBlock, SData.TVal);
                        }
                        else
                        {
                            // We must be in the first column which must be "C" attr
                            if (sBlock.CompareTo(" ") > 0)
                                sDesc = string.Format("{0}>{1}", sBlock, SData.TVal);
                            else
                                sDesc = SData.TVal;
                        }
                    } // if (SData.TAttr.StartsWith("C"))
                } // while (true)   Done scanning line m_Debug += string.Format(" \r\nDesc/amt build:{0}<< ${1}", sDesc, fAmt);
                //sDesc = sb.ToString();
                //m_Debug += string.Format("    Done with line sBlock:{0} ${1}", sBlock, fAmt);
                int iDispLn;
                string sRptLn;
                if (decAmt != 0m)
                {
                    //m_Debug += string.Format(" \r\n----->Desc/amt build:{0}<< ${1}", sDesc, fAmt);
                    m_DebugSB.Append(string.Format(" \r\n   ----->Desc/amt build:{0}<< ${1}", sDesc, decAmt));
                    iDispLn = iSub1 + 1;
                    sRptLn = string.Format("{0}:{1}", m_ReportCode, iDispLn.ToString("000"));
                    //aDesc[m_iAmtSub] = sDesc + " [" + m_ReportCode + ":" + iDispLn.ToString("000") + "]" ;
                    aDesc[m_iAmtSub] = string.Format("{0} [{1}]", sDesc, sRptLn);    // +" [" + m_ReportCode + ":" + iDisp.ToString("000") + "]";
                    adecAmt[m_iAmtSub] = decAmt;
                    asAmt[m_iAmtSub] = decAmt.ToString("F");
                    asDebit[m_iAmtSub] = " ";
                    asCredit[m_iAmtSub] = " ";
                    aCSVDescAmt[m_iAmtSub].Desc = sDesc;   // <--  7 May '17 Starting using struc for passback
                    aCSVDescAmt[m_iAmtSub].RptLn = sRptLn;   // <--               " 
                    m_iAmtSub++;
                }
            } //  for (int iSub1 = 0; iSub1 < m_iLnSub; iSub1++)
        } // private void ProcessRECRCP()


        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = =
        private void ProcessGSTSTATS()
        {
            /*
             *  File for this report is unique in that it has only one line containing all its data.
             *  
             */
            //m_Debug += "\r\nProcessGSTSTAT...";
            m_DebugSB.Append("\r\n   ProcessGSTSTATS...");

            if (m_decNetRev != 0)
            {
                //m_Debug += string.Format("\r\nDesc build:{0}<< ${1}", sDesc, fAmt) ;
                //m_DebugSB.Append(string.Format("\r\nDesc build:{0}<< ${1}", sDesc, fAmt));
                //iDisp = 1 ;
                aDesc[0] = string.Format("NET ROOM/SUITE REVENUE [{0}:001]", m_ReportCode);
                adecAmt[0] = m_decNetRev ;
                asAmt[0] = m_decNetRev.ToString("F") ;
                asDebit[0]  = " " ;
                asCredit[0] = " " ;
                aCSVDescAmt[0].Desc = "NET ROOM/SUITE REVENUE";
                aCSVDescAmt[m_iAmtSub].RptLn = string.Format("{0}:001", m_ReportCode);
                m_iAmtSub = 1;
            }
        } // private void ProcessGSTSTATS()


        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = =
        public String GetDateStr()
        {
            String sRet = "--/--/--";

            // http://www.csharp-examples.net/string-format-datetime/
            sRet = String.Format("{0:MM/dd/yy}", oReportDate);

            return sRet;
        }  // public String GetDateStr()

        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = =
        public String GetFileName()
        {
            /*
             *  Seperate file name from drive:\folder 
             * 
             */
            String sRet = "???";

            int iPos = sFile.LastIndexOf('\\');
            if (iPos >= 0)
                sRet = sFile.Substring(iPos + 1);

            return sRet;
        }  // public String GetFileName()


        //
        //  Below is redesign on parceing mechanism 
        //
        //

        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = =
        private CharType ClassChar(char inChar)
        {
            CharType CType = CharType.Other;

            if (inChar >= '0' && inChar <= '9')
                CType = CharType.Dig;
            else if (inChar >= 'A' && inChar <= 'Z' || inChar >= 'a' && inChar <= 'z')
                CType = CharType.Char;
            else if (inChar == '"') CType = CharType.DQuote;
            else if (inChar == ',') CType = CharType.Comma;
            else if (inChar == ' ') CType = CharType.Space;
            else if (inChar == '/') CType = CharType.Slash;
            else if (inChar == '-') CType = CharType.Dash;
            else if (inChar == '(') CType = CharType.LParen;
            else if (inChar == ')') CType = CharType.RParen;
            else if (inChar == '.') CType = CharType.Decimal;
            else if (inChar == '$') CType = CharType.Dollar;
            else
                CType = CharType.Char;

            return CType;

        }  // private CharType ClassChar(char inChar)

        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = =
        private string AssignTokCode(LexState inState)
        {
            string sRet = "?";
            switch (inState)
            {
                case LexState.StartToken: sRet = "x"; break;
                case LexState.AlphaNum: sRet = "A"; break;
                case LexState.Numeric: sRet = "N"; break;
                case LexState.Char: sRet = "C"; break;
                case LexState.Dash: sRet = "C"; break;
                case LexState.Date: sRet = "D"; break;
                case LexState.Neg: sRet = "G"; break;
                case LexState.Space: sRet = ""; break;

            } // switch (LState)
            return sRet;
        }  // private string AssignTokCode(LexState inState)


        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = =
        private decimal DecimalSeg(string inSeg)
        {

            /*
             * 5 VII '17
             * 
             *  Convert token to decimal 
             *    Formats ($9,999.99), $9,999.99, etc.  
             *    
             */
            
            string sDecimal;
            if (inSeg.StartsWith("("))
                sDecimal = "-" + inSeg.Substring(1, inSeg.Length - 2);
            else
                sDecimal = inSeg;

            decimal decOut = -88888.08m;
            try
            {
                decOut =  decimal.Parse(sDecimal) ;
            }
            catch (FormatException)
            {
                decOut = 999999.88m;
            }

            return decOut ;
        } // private float AmtToken(string inTok)


        // = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = = =
        private float FloatSeg(string inSeg)
        {
            // 
            //  Convert token to float 
            //     Formats ($9,999.99), $9,999.99, etc.  
            //

            string sFloat;
            if (inSeg.StartsWith("("))
                sFloat = "-" + inSeg.Substring(1, inSeg.Length - 2);
            else
                sFloat = inSeg;

            float fOut = -88888.08f;
            try
            {
                fOut = float.Parse(sFloat);
            }
            catch (FormatException)
            {
                fOut = 999999.88f;
            }

            return fOut;
        } // private float AmtToken(string inTok)



    }  // class FileCSV
}


```
