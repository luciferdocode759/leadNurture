# leadAssignmentCode.

Trigger:

trigger LeadAssignmentTrigger on Lead (before insert) {
    if (Trigger.isBefore && Trigger.isInsert) {
  LeadAssignmentHandler.assignLeads(Trigger.new);
}
    if (Trigger.isBefore && Trigger.isUpdate) {
        LeadTriggerHandler.validateLeads(Trigger.new);
    }
}

Handlerclass:

public class LeadAssignmentHandler {
    
        public static void assignLeads(List<Lead> leads) {
        // List of sales representatives
        List<User> salesReps = [SELECT Id, Name FROM User WHERE Profile.Name = 'Sales Rep'];
        Integer repCount = salesReps.size();
        
        // Variables to keep track of the round-robin assignment
        Integer currentRepIndex = 0;
        
        for (Lead lead : leads) {
           
            lead.OwnerId = salesReps[currentRepIndex].Id;
            
       
            currentRepIndex = (currentRepIndex + 1) / repCount;
        }
    }
}


LeadAssignmentHandler test class:

@isTest
public class LeadAssignmentHandlerTest {
    @isTest
  public  static void testLeadAssignment() {
       
        Profile salesRepProfile = [SELECT Id FROM Profile WHERE Name = 'Sales Rep' LIMIT 1];
        User salesRep1 = new User(Alias = 'rep1', Email='rep1@test.com', 
                                  EmailEncodingKey='UTF-8', LastName='Rep1', 
                                  LanguageLocaleKey='en_US', LocaleSidKey='en_US', 
                                  ProfileId = salesRepProfile.Id, 
                                  TimeZoneSidKey='America/Los_Angeles', 
                                  UserName='rep1@test.com');
        User salesRep2 = new User(Alias = 'rep2', Email='rep2@test.com', 
                                  EmailEncodingKey='UTF-8', LastName='Rep2', 
                                  LanguageLocaleKey='en_US', LocaleSidKey='en_US', 
                                  ProfileId = salesRepProfile.Id, 
                                  TimeZoneSidKey='America/Los_Angeles', 
                                  UserName='rep2@test.com');
        insert new List<User>{salesRep1, salesRep2};
        
        // Create test leads
        Lead lead1 = new Lead(LastName = 'Lead1', Company = 'Company1', 
                              Industry = 'Technology', LeadSource = 'Web', 
                              City = 'New York');
        Lead lead2 = new Lead(LastName = 'Lead2', Company = 'Company2', 
                              Industry = 'Finance', LeadSource = 'Email', 
                              City = 'San Francisco');
        insert new List<Lead>{lead1, lead2};
        
       
        List<Lead> leads = [SELECT Id, OwnerId FROM Lead WHERE Id IN :new List<Id>{lead1.Id, lead2.Id}];
        System.assertEquals(salesRep1.Id, leads[0].OwnerId);
        System.assertEquals(salesRep2.Id, leads[1].OwnerId);
    }
}



leadhandler apex class:

public class LeadTriggerHandler {
    
     public static void validateLeads(List<Lead> leads) {
        for (Lead lead : leads) {
            if (String.isBlank(lead.Email) || !lead.Email.contains('@')) {
                lead.addError('Lead must have a valid email address.');
            }
            if (String.isBlank(lead.Phone)) {
                lead.addError('Lead must have a valid phone number.');
            }
            if (String.isBlank(lead.Company)) {
                lead.addError('Company field cannot be blank.');
            }
           
        }
    }
}


leadHandler test class:


@isTest
public class LeadTriggerHandlerTest {
    @isTest
    static void testLeadValidation() {
        // Create a lead with missing fields
        Lead lead = new Lead(LastName = 'Test', Company = '', Email = 'invalidEmail', Phone = '');
        insert lead;

      
        lead.Company = 'Test Company';
        update lead;

        // Verify that the lead has errors
        Lead updatedLead = [SELECT Id, Email, Phone, Company FROM Lead WHERE Id = :lead.Id];
        System.assertEquals('invalidEmail', updatedLead.Email);
        System.assertEquals('', updatedLead.Phone);
        System.assertEquals('Test Company', updatedLead.Company);

        
        Test.startTest();
        try {
            update updatedLead;
            System.assert(false, 'Expected validation errors, but none were found.');
        } catch (DmlException e) {
            System.assert(e.getMessage().contains('Lead must have a valid email address.'));
            System.assert(e.getMessage().contains('Lead must have a valid phone number.'));
        }
        Test.stopTest();
    }
}
